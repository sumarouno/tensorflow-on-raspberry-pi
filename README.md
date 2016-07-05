# Installing TensorFlow on Raspberry Pi 3 (and probably 2 as well)

## Intro

_We did it!_ It took a lot of head-banging and several indirect passings-of-the-torch, but we finally got TensorFlow compiled and running properly on the Raspberry Pi! Hopefully this will enable more hardware-based machine learning projects, as well as making the distributed aspects of TensorFlow more accessible.

### Contents

* [Installing from pip (easy)](#installing-from-pip)
* [Building from source (hard)](#building-from-source)
* [Credits](#credits)
* [License](#license)

## Installing from Pip

**Note: These are unofficial binaries (though built from the minimally modified official source), and thus there is no expectation of support from the TensorFlow team. Please don't create issues for these files in the official TensorFlow repository.**

This is the easiest way to get TensorFlow onto your Raspberry Pi 3. Note that currently, the pre-built binary is targeted for Raspberry Pi 3 running Raspbian 8.0, so this may or may not work for you.

First, install the dependencies for TensorFlow:

```shell
# For Python 2.7
$ sudo apt-get install python-pip python-dev

# For Python 3.3+
$ sudo apt-get install python3-pip python3-dev
```

Next, download the wheel file from this repository and install it:

```shell
# For Python 2.7
$ wget https://github.com/samjabrahams/tensorflow-on-raspberry-pi/raw/master/bin/tensorflow-0.9-cp27-none-linux_armv7l.whl
$ sudo pip install tensorflow-0.9.0-cp27-none-linux_armv7l.whl

# For Python 3.3+
$ wget https://github.com/samjabrahams/tensorflow-on-raspberry-pi/raw/master/bin/tensorflow-0.9-py3-none-any.whl
$ sudo pip install tensorflow-0.9.0-py3-none-any.whl
```

And that should be it!

### Troubleshooting

_This section will attempt to maintain a list of remedies for problems that may occur while installing from `pip`_

#### "tensorflow-0.9-cp27-none-linux_armv7l.whl is not a supported wheel on this platform."

This wheel was built with Python 2.7, and can't be installed with a version of `pip` that uses Python 3. If you get the above message, try running the following command instead:

```
$ sudo pip2 install tensorflow-0.9-cp27-none-linux_armv7l.whl
```

Vice-versa for trying to install the Python 3 wheel. If you get the error "tensorflow-0.9-py3-none-any.whl is not a supported wheel on this platform.", try this command:

```
$ sudo pip3 install tensorflow-0.9-py3-none-any.whl
```

## Building from Source

[_Step-by-step guide_](GUIDE.md)

If you aren't able to make the wheel file from the previous section work, you may need to build from source. Additionally, if you want to use features that have not been included in an official release, such as the [initial distributed runtime](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/core/distributed_runtime), you'll have to build from source. Don't worry, as we've figured out most of the quirks of getting it right. The guide will be updated as needed to be as correct as possible.

See the [step-by-step guide here](GUIDE.md). **Warning: it takes a while.**

## Credits

_Or: people who did most of the actual work._

While we may have just gotten RPi TensorFlow compiled properly in the last few days (with the most recent grunt work done by myself and @petewarden), this effort has been going on for almost as long as TensorFlow has been open-source, and involves work that spans multiple months in separate codebases. This is an incomprehensive list of people and their work I ran across while working on this.

The majority of the source-building guide is a modified version of [these instructions for compiling TensorFlow on a Jetson TK1](http://cudamusing.blogspot.com/2015/11/building-tensorflow-for-jetson-tk1.html). Massimiliano, you are the real MVP.

@vmayoral put a huge amount of time and effort trying to put together the pieces to build TensorFlow, and was the first to get something close to a working binary.

A bunch of awesome Googlers working in both the TensorFlow and Bazel repositories helped make this possible. In no particular order: @vrv, @damienmg, @petewarden, @danbri, @ulfjack, @girving, and @nlothian

_Issue threads of interest:_

* [Initial issue for building Bazel on ARMv7l](https://github.com/bazelbuild/bazel/issues/606)
* [First thread about running TensorFlow on RPi](https://github.com/tensorflow/tensorflow/issues/254)
* [Parallel thread on building TensorFlow on ARMv7l](https://github.com/tensorflow/tensorflow/issues/445)
	* This is where the most recent conversation is located

## License

The file TENSORFLOW_LICENSE applies to any and all files in the `bin` directory, which are compiled Python wheels for TensorFlow.

Subdirectories contained within the `third_party` directory each contain relevant licenses for the code and software within those subdirectories.

The file LICENSE applies to other files in this repository. I want to stress that a majority of the lines of code found in the guide of this repository was created by others. If any of those original authors want more prominent attribution, please contact me and we can figure out how to make it acceptable.

---

_[Back to top](#installing-tensorflow-on-raspberry-pi-3-and-probably-2-as-well)_
