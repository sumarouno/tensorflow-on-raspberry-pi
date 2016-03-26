# Building TensorFlow for Raspberry Pi: a Step-By-Step Guide

_[Back to readme](README.md)_

## What You Need

* Raspberry Pi 3 Model B (probably works for RPi 2, but not verified yet)
* Internet connection to the Raspberry Pi
* A USB memory drive that can be installed as swap memory (if it is a flash drive, make sure you don't care about the drive). Anything over 1 GB should be fine
* A fair amount of time

## Overview

These instructions were crafted for a [Raspberry Pi 3 Model B](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/) running a vanilla copy of Raspbian 8.0 (jessie). It appears to work on Raspberry Pi 2, but [there are some kinks that are being worked out](https://github.com/tensorflow/tensorflow/issues/445#issuecomment-196021885). If these instructions work for different distributions, let me know!

Here's the basic plan: build a 32-bit version of [Protobuf](https://github.com/google/protobuf), use that to build a RPi-friendly version of [Bazel](https://github.com/bazelbuild/bazel), and finally use Bazel to build TensorFlow.

### Contents

1. [Install basic dependencies](#1-install-basic-dependencies)
2. [Build Protobuf](#2-build-protobuf)
3. [Build Bazel](#3-build-bazel)
4. [Install USB Memory as Swap](#4-install-a-memory-drive-as-swap-for-compiling)
5. [Compiling TensorFlow](#5-compiling-tensorflow)
	* [Building the Distributed Runtime](#55-building-the-distributed-runtime)
6. [Cleaning Up](#6-cleaning-up)
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

For Bazel:

```shell
sudo apt-get install pkg-config zip g++ zlib1g-dev unzip
```

For TensorFlow:

```
sudo apt-get install python-pip python-numpy swig python-dev
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

After following these steps, you'll have two spiffy new files: `/usr/bin/protoc` and `protobuf/java/core/target/protobuf-java-3.0.0-beta2.jar`

### 3. Build Bazel

First, move out of the `protobuf/java` directory and clone Bazel's repository.

```shell
cd ../..
git clone https://github.com/bazelbuild/bazel.git
```

Next, go into the new `bazel` direcotry and immediately checkout version 0.1.4 of Bazel. _Note: we do this because a hack-y workaround we do later on doesn't work for the most recent version of Bazel. If you have instructions for building Bazel 2.0 or later on Raspberry Pi, please send a pull request!_

```shell
cd bazel
git checkout tags/0.1.4
```

After that, copy the two Protobuf files mentioned earlier into the Bazel project. Note the naming of the files in this step- it must be precise.

```shell
sudo cp /usr/bin/protoc third_party/protobuf/protoc-linux-arm32.exe
sudo cp ../protobuf/java/core/target/protobuf-java-3.0.0-beta-2.jar third_party/protobuf/protobuf-java-3.0.0-beta-1.jar
```

Now we can build Bazel! _Note: this also takes some time._

```shell
./compile.sh
```

When the build finishes, you end up with a new binary, `output/bazel`. Copy that to your `/usr/local/bin` directory.

```shell
sudo mkdir /usr/local/bin
sudo cp output/bazel /usr/local/bin/bazel
```

To make sure it's working properly, run `bazel` on the command line and verify it prints help text.

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

### 4. Install a Memory Drive as Swap for Compiling

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

### 5. Compiling TensorFlow

First things first, clone the TensorFlow repository and move into the newly created directory.

```shell
git clone --recurse-submodules https://github.com/tensorflow/tensorflow
cd tensorflow
```

Once in the directory, we have to write a nifty one-liner that is incredibly important. The next line goes through all files and changes references of 64-bit program implementations (which we don't have access to) to 32-bit implementations. Neat!

```shell
grep -Rl "lib64"| xargs sed -i 's/lib64/lib/g'
```

And surprisingly, that's all we need to do! There were a couple bugs that needed workaround, but as of now they are gone. If things get wonky, I'll update this file to reflect any changes you need to make.

Let's first configure Bazel:

```shell
$ ./configure

Please specify the location of python. [Default is /usr/bin/python]: /usr/bin/python
Do you wish to build TensorFlow with GPU support? [y/N] N
```

Now we can use it to build TensorFlow! **Warning: This takes a really, really long time. Several hours.**

```shell
bazel build -c opt --local_resources 1024,1.0,1.0 --verbose_failures --copt="-mfpu=neon" tensorflow/tools/pip_package:build_pip_package
```

_Note: I toyed around with telling Bazel to use all four cores in the Raspberry Pi, but that seemed to make compiling more prone to completely locking up. This process takes a long time regardless, so I'm sticking with the more reliable options here. If you want to be bold, try using `--local_resources 1024,2.0,1.0` or `--local_resources 1024,4.0,1.0`_

When you wake up the next morning and it's finished compiling, you're in the home stretch! Use the built binary file to create a Python wheel.

```shell
bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg
```

And then install it!

```shell
sudo pip install /tmp/tensorflow_pkg/tensorflow-0.7.1-cp27-none-linux_armv7l.whl
```

### 5.5 Building the Distributed Runtime

If all has gone according to plan, you should be the proud owner of a TensorFlow-capable Raspberry Pi! But before removing your swap drive, you may want to consider building one of the optional TensorFlow components, such as the [initial distributed runtime](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/core/distributed_runtime). Building it is a breeze (relatively):

```shell
bazel build -c opt --local_resources 1024,1.0,1.0 --verbose_failures --copt="-mfpu=neon" tensorflow/core/distributed_runtime/rpc:grpc_tensorflow_server
```

And then test it out by running a local GRPC server:

```
$ bazel-bin/tensorflow/core/distributed_runtime/rpc/grpc_tensorflow_server --cluster_spec='local|localhost:2222' --job_name=local --task_id=0

I tensorflow/core/distributed_runtime/rpc/grpc_tensorflow_server.cc:74] Peer local 1 {localhost:2222}
I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:199] Initialize HostPortsGrpcChannelCache for job local -> {localhost:2222}
I tensorflow/core/distributed_runtime/rpc/grpc_server_lib.cc:203] Started server with target: grpc://localhost:2222
```

### 6. Cleaning Up

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