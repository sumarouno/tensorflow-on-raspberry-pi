# Inception-v3 speed: Raspberry Pi 3

_Latest update: December 1, 2016; TensorFlow 0.11.0_

## About

This file contains some very basic run-time statistics for [TensorFlow's pre-trained Inception-v3 model](https://www.tensorflow.org/versions/r0.7/tutorials/image_recognition/index.html) running on a [Raspberry Pi 3 Model B](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/) as compared to an [Early 2013 15 inch Retina MacBook Pro](https://support.apple.com/kb/SP669?locale=en_US) with an Intel i7-3740QM CPU as well as a desktop rig running Ubuntu 14.04 with a Titan X Maxwell GPU and Intel i7-5820K CPU.

To run this benchmark, I use a modified version of the example [classify_image.py script](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/models/image/imagenet). I made minor modifications to collect and print out run-time information after processing. The modified file is available here: [classify\_image\_timed.py](classify_image_timed.py).

## Summary

* _warmup_runs_ refers to the number of calls to `Session.run` before starting the benchmarking in order to "warmup" the model. TensorFlow makes adjustments on the fly, so the first few times running the model are slower than subsequent runs
* A _run_ is the time between the start of a call to `Session.run` and when it returns. We list the best, worst, and average time (averaged over 25 runs)
* _Build_ is the amount of time spent constructing the Inception model from the protobuf file.

<table>

	<tr>
		<th colspan="6"><i>TensorFlow version 0.11.0</i></th>
	</tr>

	<tr>
		<td></td>
		<th><i>Model</i></th>
		<th><i>Best run (sec)</i></th>
		<th><i>Worst run (sec)</i></th>
		<th><i>Average run (sec)</i></th>
		<th><i>Build time(sec)</i></th>
	</tr>

	<tr>
		<th rowspan="4"><b>warmup_runs=10</b></th>
		<td><b>Raspberry Pi 3</b></td>
		<td><b>1.8646</b></td>
		<td><b>2.1782</b></td>
		<td><b>1.9805</b></td>
		<td><b>4.8962</b></td>
	</tr>

	<tr>
		<td>Intel i7-3740QM (Early 2013 MacBook Pro)</td>
		<td>0.2146</td>
		<td>0.2425</td>
		<td>0.2272</td>
		<td>1.3104</td>
	</tr>

	<tr>
		<td>Intel i7-5820K (Ubuntu 14.04)</td>
		<td>0.1397</td>
		<td>0.1730</td>
		<td>0.1567</td>
		<td>0.7064</td>
	</tr>

	<tr>
		<td>NVIDIA Titan X (Maxwell), Intel i7-5820K (Ubuntu 14.04)</td>
		<td>0.0240</td>
		<td>0.0290</td>
		<td>0.0259</td>
		<td>0.9566</td>
	</tr>

	<tr>
		<th rowspan="4"><b>warmup_runs=0</b></th>
		<td><b>Raspberry Pi 3</b></td>
		<td><b>1.8541</b></td>
		<td><b>6.3338</b></td>
		<td><b>2.0656</b></td>
		<td><b>4.9755</b></td>
	</tr>

	<tr>
		<td>Intel i7-3740QM (Early 2013 Retina MacBook Pro)</td>
		<td>0.2174</td>
		<td>1.3151</td>
		<td>0.2662</td>
		<td>1.2761</td>
	</tr>

	<tr>
		<td>Intel i7-5820K (Ubuntu 14.04)</td>
		<td>0.1435</td>
		<td>0.7027</td>
		<td>0.1750</td>
		<td>0.7103</td>
	</tr>

	<tr>
		<td>NVIDIA Titan X (Maxwell), Intel i7-5820K (Ubuntu 14.04)</td>
		<td>0.0232</td>
		<td>1.5800</td>
		<td>0.0871</td>
		<td>0.7659</td>
	</tr>

</table>

### Remarks

* Test performance has gotten significantly better over the past several releases of TensorFlow, though running Inception on a Raspberry Pi still takes longer than a second when using Python
* Warming up your `Session` is _crucial_. There have been many issues opened in this repo asking how to improve performance, so here's the number one thing to start with: keep your `Session` persistent to take advantage of automatic optimization tweaks.
* Along the same lines: do _not_ simply call your Python script from bash every time you want to classify an image. It takes multiple seconds to rebuild the Inception graph from scratch, which can slow down your model by multiple times (this test doesn't include the time it takes to import `tensorflow`, which is another thing to benchmark...). This goes for pretty much any TensorFlow model you use- keep some sort of rudimentary server running that can respond to requests and utilize a live TensorFlow `Session`
* Running the [TensorFlow benchmark tool](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/tools/benchmark) shows sub-second (~500-600ms) average run times for the Raspberry Pi (I'll need to do another write-up with more details). Since this benchmark is run entirely in C++, we'd expect it to run faster than through Python. The question is whether or not all ~1.5 seconds of difference between these tests is entirely due to the communication layer between Python and the C++ core.

## About `classify_image_timed.py`

I add two additional flags to `classify_image_timed.py` which allow users to easily change the number of test runs (runs that will collect information), as well as the number of "warmup" runs used. Simply pass in a number to `--num_runs` or `--warmup_runs` when calling the script:

```bash
# Use a sample size of 100 runs
$ python classify_image_timed.py --num_runs=100

# Don't include any warmup runs
$ python classify_image_timed.py --warmup_runs=0
```
