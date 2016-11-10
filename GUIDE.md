# Building TensorFlow for Raspberry Pi: a Step-By-Step Guide

_[Back to readme](README.md)_

## What You Need

* Raspberry Pi 2 or 3 Model B
* An SD card running Raspbian with several GB of free space
	* An 8 GB card with a fresh install of Raspbian **does not** have enough space. A 16 GB SD card minimum is recommended.
	* These instructions may work on Linux distributions other than Raspbian
* Internet connection to the Raspberry Pi
* A USB memory drive that can be installed as swap memory (if it is a flash drive, make sure you don't care about the drive). Anything over 1 GB should be fine
* A fair amount of time

## Overview

These instructions were crafted for a [Raspberry Pi 3 Model B](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/) running a vanilla copy of Raspbian 8.0 (jessie). It appears to work on Raspberry Pi 2, but [there are some kinks that are being worked out](https://github.com/tensorflow/tensorflow/issues/445#issuecomment-196021885). If these instructions work for different distributions, let me know!

Here's the basic plan: build a 32-bit version of [Protobuf](https://github.com/google/protobuf), use that to build a RPi-friendly version of [Bazel](https://github.com/bazelbuild/bazel), and finally use Bazel to build TensorFlow.

### Contents

1. [Install basic dependencies](#1-install-basic-dependencies)
2. [Build Protobuf](#2-build-protobuf)
3. [Build gRPC](#3-build-grpc)
3. [Build Bazel](#4-build-bazel)
4. [Install USB Memory as Swap](#5-install-a-memory-drive-as-swap-for-compiling)
5. [Compiling TensorFlow](#6-compiling-tensorflow)
6. [Cleaning Up](#7-cleaning-up)
7. [References](#references)

## The Build

### 1. Install basic dependencies

First, update apt-get to make sure it knows where to download everything.

```shell
sudo apt-get update
```

Next, install some base dependencies and tools we'll need later.

For Protobuf:

```
sudo apt-get install autoconf automake libtool maven
```

For gRPC:

```
sudo apt-get install oracle-java7-jdk
# Select the jdk-7-oracle option for the update-alternatives command
sudo update-alternatives --config java
```

For Bazel:

```shell
sudo apt-get install pkg-config zip g++ zlib1g-dev unzip
```

For TensorFlow:

```
# For Python 2.7
sudo apt-get install python-pip python-numpy swig python-dev
sudo pip install wheel

# For Python 3.3+
sudo apt-get install python3-pip python3-numpy swig python3-dev
sudo pip3 install wheel
```

To be able to take advantage of certain optimization flags:

```
sudo apt-get install gcc-4.8 g++-4.8
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 100
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.8 100
```

Finally, for cleanliness, make a directory that will hold the Protobuf, Bazel, and TensorFlow repositories.

```shell
mkdir tf
cd tf
```

### 2. Build Protobuf

Clone the Protobuf repository.

```shell
git clone https://github.com/google/protobuf.git
```

Now move into the new `protobuf` directory, configure it, and `make` it. _Note: this takes a little while._

```shell
cd protobuf
git checkout v3.0.0-beta-3.3
./autogen.sh
./configure --prefix=/usr
make -j 4
sudo make install
```

Once it's made, we can move into the `java` directory and use Maven to build the project.

```shell
cd java
mvn package
```

After following these steps, you'll have two spiffy new files: `/usr/bin/protoc` and `protobuf/java/core/target/protobuf-java-3.0.0-beta3.jar`

### 3. Build gRPC

Next, we need to build [gRPC-Java](https://github.com/grpc/grpc-java), the Java implementation of [gRPC](http://www.grpc.io/). Move out of the `protobuf/java` directory and clone gRPC's repository.

```shell
cd ../..
git clone https://github.com/grpc/grpc-java.git
```

```shell
cd grpc-java
git checkout v0.14.1
```

```shell
cd compiler
nano build.gradle
```

Around line 47:

```
gcc(Gcc) {
	target("linux_arm-v7") {
		cppCompiler.executable = "/usr/bin/gcc"
	}
}
```

Around line 60, add section for `'linux_arm-v7'`:

```
...
	x86_64 {
		architecture "x86_64"
	}
	'linux_arm-v7' {
		architecture "arm32"
		operatingSystem "linux"
	}
```

Around line 64, add `'arm32'` to list of architectures:

```
...
components {
	java_plugin(NativeExecutableSpec) {
			if (arch in ['x86_32', 'x86_64', 'arm32'])
...
```

Around line 148, replace content inside of `protoc` section to hard code path to `protoc` binary:

```
protoc {
	path = '/usr/bin/protoc'
}
```

Once all of that is taken care of, run this command to build gRPC:

```shell
../gradlew java_pluginExecutable
```

### 4. Build Bazel

First, move out of the `grpc-java/compiler` directory and clone Bazel's repository.

```shell
cd ../..
git clone https://github.com/bazelbuild/bazel.git
```

Next, go into the new `bazel` directory and immediately checkout version 0.3.1 of Bazel.

```shell
cd bazel
git checkout 0.3.2
```

After that, copy the generated Protobuf and gRPC files we created earlier into the Bazel project. Note the naming of the files in this step- it must be precise.

```shell
sudo cp /usr/bin/protoc third_party/protobuf/protoc-linux-arm32.exe
sudo cp ../protobuf/java/core/target/protobuf-java-3.0.0-beta-3.jar third_party/protobuf/protobuf-java-3.0.0-beta-1.jar
sudo cp ../grpc-java/compiler/build/exe/java_plugin/protoc-gen-grpc-java third_party/grpc/protoc-gen-grpc-java-0.15.0-linux-x86_32.exe
```

Before building Bazel, we need to set the `javac` maximum heap size for this job, or else we'll get an OutOfMemoryError. To do this, we need to make a small addition to `bazel/scripts/bootstrap/compile.sh`. (Shout-out to @SangManLINUX for [pointing this out.](https://github.com/samjabrahams/tensorflow-on-raspberry-pi/issues/5#issuecomment-210965695).

```shell
nano scripts/bootstrap/compile.sh
```

Around line 46, you'll find this code:

```shell
if [ "${MACHINE_IS_64BIT}" = 'yes' ]; then
	PROTOC=${PROTOC:-third_party/protobuf/protoc-linux-x86_64.exe}
	GRPC_JAVA_PLUGIN=${GRPC_JAVA_PLUGIN:-third_party/grpc/protoc-gen-grpc-java-0.15.0-linux-x86_64.exe}
else
	if [ "${MACHINE_IS_ARM}" = 'yes' ]; then
		PROTOC=${PROTOC:-third_party/protobuf/protoc-linux-arm32.exe}
	else
		PROTOC=${PROTOC:-third_party/protobuf/protoc-linux-x86_32.exe}
		GRPC_JAVA_PLUGIN=${GRPC_JAVA_PLUGIN:-third_party/grpc/protoc-gen-grpc-java-0.15.0-linux-x86_32.exe}
	fi
fi
```

Change it to the following:

```shell
if [ "${MACHINE_IS_64BIT}" = 'yes' ]; then
	PROTOC=${PROTOC:-third_party/protobuf/protoc-linux-x86_64.exe}
	GRPC_JAVA_PLUGIN=${GRPC_JAVA_PLUGIN:-third_party/grpc/protoc-gen-grpc-java-0.15.0-linux-x86_64.exe}
else
	PROTOC=${PROTOC:-third_party/protobuf/protoc-linux-arm32.exe}
	GRPC_JAVA_PLUGIN=${GRPC_JAVA_PLUGIN:-third_party/grpc/protoc-gen-grpc-java-0.15.0-linux-linux-arm32.exe}
fi
```


Move down to line 149, where you'll see the following block of code:

```shell
run "${JAVAC}" -classpath "${classpath}" -sourcepath "${sourcepath}" \
      -d "${output}/classes" -source "$JAVA_VERSION" -target "$JAVA_VERSION" \
      -encoding UTF-8 "@${paramfile}"
```

At the end of this block, add in the `-J-Xmx500M` flag, which sets the maximum size of the Java heap to 500 MB:

```
run "${JAVAC}" -classpath "${classpath}" -sourcepath "${sourcepath}" \
      -d "${output}/classes" -source "$JAVA_VERSION" -target "$JAVA_VERSION" \
      -encoding UTF-8 "@${paramfile}" -J-Xmx500M
```

Next up, we need to adjust `third_party/protobuf/BUILD` - open it up in your text editor.

```
nano third_party/protobuf/BUILD
```

We need to add this last line around line 29:

```shell
...
	"//third_party:freebsd": ["protoc-linux-x86_32.exe"],
	"//third_party:arm": ["protoc-linux-arm32.exe"],
}),
...
```

Finally, we have to add one thing to `tools/cpp/cc_configure.bzl` - open it up for editing:

```shell
nano tools/cpp/cc_configure.bzl
```

And place this in around line 141 (at the beginning of the `_get_cpu_value` function):

```shell
...
"""Compute the cpu_value based on the OS name."""
return "arm"
...
```

Now we can build Bazel! _Note: this also takes some time._

```shell
sudo ./compile.sh
```

When the build finishes, you end up with a new binary, `output/bazel`. Copy that to your `/usr/local/bin` directory.

```shell
sudo mkdir /usr/local/bin
sudo cp output/bazel /usr/local/bin/bazel
```

To make sure it's working properly, run `bazel` on the command line and verify it prints help text. Note: this may take 15-30 seconds to run, so be patient!

```shell
$ bazel

Usage: bazel <command> <options> ...

Available commands:
  analyze-profile     Analyzes build profile data.
  build               Builds the specified targets.
  canonicalize-flags  Canonicalizes a list of bazel options.
  clean               Removes output files and optionally stops the server.
  dump                Dumps the internal state of the bazel server process.
  fetch               Fetches external repositories that are prerequisites to the targets.
  help                Prints help for commands, or the index.
  info                Displays runtime info about the bazel server.
  mobile-install      Installs targets to mobile devices.
  query               Executes a dependency graph query.
  run                 Runs the specified target.
  shutdown            Stops the bazel server.
  test                Builds and runs the specified test targets.
  version             Prints version information for bazel.

Getting more help:
  bazel help <command>
                   Prints help and options for <command>.
  bazel help startup_options
                   Options for the JVM hosting bazel.
  bazel help target-syntax
                   Explains the syntax for specifying targets.
  bazel help info-keys
                   Displays a list of keys used by the info command.
```

Move out of the `bazel` directory, and we'll move onto the next step.

```shell
cd ..
```

### 5. Install a Memory Drive as Swap for Compiling

In order to succesfully build TensorFlow, your Raspberry Pi needs a little bit more memory to fall back on. Fortunately, this process is pretty straightforward. Grab a USB storage drive that has at least 1GB of memory. I used a flash drive I could live without that carried no important data. That said, we're only going to be using the drive as swap while we compile, so this process shouldn't do too much damage to a relatively new USB drive.

First, put insert your USB drive, and find the `/dev/XXX` path for the device.

```shell
sudo blkid
```

As an example, my drive's path was `/dev/sda1`

Once you've found your device, unmount it by using the `umount` command.

```shell
sudo umount /dev/XXX
```

Then format your device to be swap:

```shell
sudo mkswap /dev/XXX
```

If the previous command outputted an alphanumeric UUID, copy that now. Otherwise, find the UUID by running `blkid` again. Copy the UUID associated with `/dev/XXX`

```shell
sudo blkid
```

Now edit your `/etc/fstab` file to register your swap file. (I'm a Vim guy, but Nano is installed by default)

```shell
sudo nano /etc/fstab
```

On a separate line, enter the following information. Replace the X's with the UUID (without quotes)

```bash
UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX none swap sw,pri=5 0 0
```

Save `/etc/fstab`, exit your text editor, and run the following command:

```shell
sudo swapon -a
```

If you get an error claiming it can't find your UUID, go back and edit `/etc/fstab`. Replace the `UUID=XXX..` bit with the original `/dev/XXX` information.

```shell
sudo nano /etc/fstab
```

```bash
# Replace the UUID with /dev/XXX
/dev/XXX none swap sw,pri=5 0 0
```

Alright! You've got swap! Don't throw out the `/dev/XXX` information yet- you'll need it to remove the device safely later on.

### 6. Compiling TensorFlow

First things first, clone the TensorFlow repository and move into the newly created directory.

```shell
git clone --recurse-submodules https://github.com/tensorflow/tensorflow
cd tensorflow
```

_Note: if you're looking to build to a specific version or commit of TensorFlow (as opposed to the HEAD at master), you should `git checkout` it now._

Once in the directory, we have to write a nifty one-liner that is incredibly important. The next line goes through all files and changes references of 64-bit program implementations (which we don't have access to) to 32-bit implementations. Neat!

```shell
grep -Rl 'lib64' | xargs sed -i 's/lib64/lib/g'
```

Next, we need to delete a particular line in `tensorflow/core/platform/platform.h`. Open up the file in your favorite text editor:

```shell
$ sudo nano tensorflow/core/platform/platform.h
```

Now, scroll down toward the bottom and delete the following line containing `#define IS_MOBILE_PLATFORM`:

```cpp
#elif defined(__arm__)
#define PLATFORM_POSIX
...
#define IS_MOBILE_PLATFORM   <----- DELETE THIS LINE
```

This keeps our Raspberry Pi device (which has an ARM CPU) from being recognized as a mobile device.

Now let's configure Bazel:

```shell
$ ./configure

Please specify the location of python. [Default is /usr/bin/python]: /usr/bin/python
Do you wish to build TensorFlow with Google Cloud Platform support? [y/N] N
Do you wish to build TensorFlow with GPU support? [y/N] N
```

_Note: if you want to build for Python 3, specify `/usr/bin/python3` for Python's location._

Now we can use it to build TensorFlow! **Warning: This takes a really, really long time. Several hours.**

```shell
bazel build -c opt --copt="-mfpu=neon-vfpv4" --copt="-funsafe-math-optimizations" --copt="-ftree-vectorize" --local_resources 1024,1.0,1.0 --verbose_failures tensorflow/tools/pip_package:build_pip_package
```

_Note: I toyed around with telling Bazel to use all four cores in the Raspberry Pi, but that seemed to make compiling more prone to completely locking up. This process takes a long time regardless, so I'm sticking with the more reliable options here. If you want to be bold, try using `--local_resources 1024,2.0,1.0` or `--local_resources 1024,4.0,1.0`_

When you wake up the next morning and it's finished compiling, you're in the home stretch! Use the built binary file to create a Python wheel.

```shell
bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg
```

And then install it!

```shell
sudo pip install /tmp/tensorflow_pkg/tensorflow-0.10-cp27-none-linux_armv7l.whl
```

### 7. Cleaning Up

There's one last bit of house-cleaning we need to do before we're done: remove the USB drive that we've been using as swap.

First, turn off your drive as swap:

```shell
sudo swapoff /dev/XXX
```

Finally, remove the line you wrote in `/etc/fstab` referencing the device

```
sudo nano /etc/fstab
```

Then reboot your Raspberry Pi.

**And you're done!** You deserve a break.

## References

1. [Building TensorFlow for Jetson TK1](http://cudamusing.blogspot.com/2015/11/building-tensorflow-for-jetson-tk1.html)
2. [Turning a USB Drive into swap](http://askubuntu.com/questions/173676/how-to-make-a-usb-stick-swap-disk)
3. [Safely removing USB swap space](http://askubuntu.com/questions/616437/is-it-safe-to-delete-a-swap-partition-on-a-usb-install)

---

_[Back to top](#building-tensorflow-for-raspberry-pi-a-step-by-step-guide)_
