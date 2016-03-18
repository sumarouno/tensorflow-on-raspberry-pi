# Inception-v3 speed: Raspberry Pi 3 vs 2013 MacBook Pro

## About

This file contains some very basic run-time statistics for [TensorFlow's pre-trained Inception-v3 model](https://www.tensorflow.org/versions/r0.7/tutorials/image_recognition/index.html) running on a [Raspberry Pi 3 Model B](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/) as compared to an [Early 2013 15 inch Retina MacBook Pro](https://support.apple.com/kb/SP669?locale=en_US).

Out of the box, Inception-v3 is available to run from either a Python script [(tensorflow/models/image/imagenet/classify_image.py)](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/models/image/imagenet) or as a compiled C++ binary [(tensorflow/examples/label_image/main.cc)](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/examples/label_image). To get the rough benchmarks used in this file, I made minor modifications in both files to print out run-time information after processing. The modified files are available in this directory: [classify\_image\_timed.py](classify_image_timed.py) and [main.cc](main.cc) Both tests used the default grace_hopper.jpg image used in the Inception-v3 C++ file. 

## Summary

<table>

	<tr>
		<td></td>
		<th colspan="3">Python (CPU)</th>
		<th colspan="3">C++ (CPU)</th>
	</tr>
	
	<tr>
		<td><i>Model</i></td>
		<td><i>Build (sec)</i></td>
		<td><i>Eval (sec)</i></td>
		<td><i>Total (sec)</i></td>
		<td><i>Build (sec)</i></td>
		<td><i>Eval (sec)</i></td>
		<td><i>Total (sec)</i></td>
	</tr>
	
	<tr>
		<td>Raspberry Pi 3</td>
		<td>3.496</td>
		<td>11.004</td>
		<td>14.500</td>
		<td>0.436</td>
		<td>6.969</td>
		<td>7.405</td>
	</tr>
	
	<tr>
		<td>2013 MacBook Pro</td>
		<td>0.747</td>
		<td>1.421</td>
		<td>2.168</td>
		<td>0.253</td>
		<td>6.036</td>
		<td>6.289</td>
	</tr>
	
	<tr>
		<td>Time increase on Raspberry Pi</td>
		<td>4.68x</td>
		<td><b>7.744x</b></td>
		<td>6.688x</td>
		<td>1.723x</td>
		<td><b>1.155x</b></td>
		<td>1.177x</td>
	</tr>
	
</table>

### Remarks

* The good-ish news: the RPi3 appears to achieve fair performance relative to the MacBook Pro when running the compiled C++ binary
* The bad news: The Python version is **really** slow. From just this test, I can't tell if the Python bindings to C++ aren't working properly, but I think it's definitely worth looking into
	* Dan Brickley (@danbri) shared some results when [testing out a camera module on his Raspberry Pi 3](https://twitter.com/danbri/status/709903532216995842). Direct link to Gist [here](https://gist.githubusercontent.com/danbri/ee6323d78ca14e616e4e/raw/6f50a897a59cb25d6c5e8f43fdfb0392fe9945d8/gistfile1.txt)
	* Pete Warden (@petewarden) mentioned that the compiler [may not be using NEON](https://github.com/tensorflow/tensorflow/issues/445#issuecomment-196021885) on the Raspberry Pi 2 while attempting to build TensorFlow. While my tests did not take a minute to run, @danbri's results suggest similar performance to @petewarden's; this may be a first place to look for improvements
* On Mac, the Python version appears to run _much_ faster than the C++ binary. I'm not quite sure how this happened. I'd like to test on other systems to see if the results hold.

## Outputs

### Python

#### Raspberry Pi 3, Raspbian 8.0
```
giant panda, panda, panda bear, coon bear, Ailuropoda melanoleuca (score = 0.89233)
indri, indris, Indri indri, Indri brevicaudatus (score = 0.00859)
lesser panda, red panda, panda, bear cat, cat bear, Ailurus fulgens (score = 0.00264)
custard apple (score = 0.00141)
earthstar (score = 0.00107)
Build graph time: 3.495808
Eval time: 11.004332
```

#### Early 2013, 15-inch MacBook Pro (2.7 GHz Intel Core i7), OS X 10.11.1

```
giant panda, panda, panda bear, coon bear, Ailuropoda melanoleuca (score = 0.89233)
indri, indris, Indri indri, Indri brevicaudatus (score = 0.00859)
lesser panda, red panda, panda, bear cat, cat bear, Ailurus fulgens (score = 0.00264)
custard apple (score = 0.00141)
earthstar (score = 0.00107)
Build graph time: 0.746465
Eval time: 1.421328
```

---

### C++

#### Raspberry Pi 3, Raspbian 8.0
```
I tensorflow/examples/label_image/main.cc:210] military uniform (866): 0.647298
I tensorflow/examples/label_image/main.cc:210] suit (794): 0.0477194
I tensorflow/examples/label_image/main.cc:210] academic gown (896): 0.0232409
I tensorflow/examples/label_image/main.cc:210] bow tie (817): 0.0157354
I tensorflow/examples/label_image/main.cc:210] bolo tie (940): 0.0145024

# First time running after booting system:
4450 milliseconds to build graph
7005 milliseconds to evaluate image

# Subsequent time
436 milliseconds to build graph
6969 milliseconds to evaluate image
```

#### Early 2013, 15-inch MacBook Pro (2.7 GHz Intel Core i7), OS X 10.11.1

```
I tensorflow/examples/label_image/main.cc:210] military uniform (866): 0.647299
I tensorflow/examples/label_image/main.cc:210] suit (794): 0.0477195
I tensorflow/examples/label_image/main.cc:210] academic gown (896): 0.0232407
I tensorflow/examples/label_image/main.cc:210] bow tie (817): 0.0157355
I tensorflow/examples/label_image/main.cc:210] bolo tie (940): 0.0145023

# First running time after booting system:
468 milliseconds to build graph
6124 milliseconds to evaluate image

# Subsequent running times
253 milliseconds to build graph
6036 milliseconds to evaluate image
```