## Writeup Template
### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---

**Vehicle Detection Project**

The goals / steps of this project are the following:

* Perform a Histogram of Oriented Gradients (HOG) feature extraction on a labeled training set of images and train a classifier Linear SVM classifier
* Optionally, you can also apply a color transform and append binned color features, as well as histograms of color, to your HOG feature vector.
* Note: for those first two steps don't forget to normalize your features and randomize a selection for training and testing.
* Implement a sliding-window technique and use your trained classifier to search for vehicles in images.
* Run your pipeline on a video stream (start with the test_video.mp4 and later implement on full project_video.mp4) and create a heat map of recurring detections frame by frame to reject outliers and follow detected vehicles.
* Estimate a bounding box for vehicles detected.

[//]: # (Image References)
[image1]: ./examples/car_not_car.png
[blurred]: ./writeup-images/blurred.png
[boxes]: ./writeup-images/boxes.png
[boxes_cropped]: ./writeup-images/boxes_cropped.png
[cropped]: ./writeup-images/cropped.png
[heatmap]: ./writeup-images/heatmap.png
[labeled]: ./writeup-images/labeled.png
[original]: ./writeup-images/original.png
[scaled]: ./writeup-images/scaled.png
[video1]: ./project_video.mp4

## [Rubric](https://review.udacity.com/#!/rubrics/513/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation. 

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Vehicle-Detection/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point. 

You're reading it!

### Histogram of Oriented Gradients (HOG)

#### 1. Explain how (and identify where in your code) you extracted HOG features from the training images.

Instead of using the HOG function provided by `cv2`, I used a Convolutional Neural Network.  The HOG features (or at least hog like features) are extracted through the first few layers of the neural network.

_the entire network definition can be found in the `create_model()` function of the notebook_

<insert visualization of the network>* (I tried to get a visualization of the network, but it was taking too long to figure out.)

I started by reading in all the `vehicle` and `non-vehicle` locations as a `glob`.  Then, iterated over that glob and loaded the images, converted them to RGB space and augmented them by flipping the images and randomly zooming in on part of the image in order to allow for more robust training and to allow the network to work well on different sized cars.

_this augmentation can be found in the `agument()` function in the notebook.  (yes, I know I misspelled it)_


#### 2. Explain how you settled on your final choice of HOG parameters.

The Network learns these parameters it's self through gradient decent.  And, regarding how I designed my network, I would like to say that my network design has some philosophy behind it. However, the truth is I tried various combinations of hyperparameters and layers and this one just worked the best.  It should be noted that I did **only** used Conv2D, MaxPooling2D and Dropout layers, all of which can take in a variable sized input.

#### 3. Describe how (and identify where in your code) you trained a classifier using your selected HOG features (and color features if you used them).

_this can be found under the "Train Model" header of the notebook_

To train the network in keras it's pretty straight forward, I used a Mean Squared Error and an Adam Optimizer and ran it 10 times over the entire dataset (including the augmented images).

### Sliding Window Search

#### 1. Describe how (and identify where in your code) you implemented a sliding window search.  How did you decide what scales to search and how much to overlap windows?

After training the images on the 64x64x3 training images, I then ran it over the entire region of interest in the image.  The reason I can do this is because all of my layers allowed for variable sized input.  This has the nice feature that instead of outputting a single number for the prediction it outputs a heatmap directly for what the predictions are across the image.

So, the "Sliding window" is done by allowing the Convolutions to run over the entire image.

#### 2. Show some examples of test images to demonstrate how your pipeline is working.  What did you do to optimize the performance of your classifier?

### Pipeline (single image)

To start with I loaded a image from `./test_images/`

![original][]

#### 1. Crop

The first step is to crop to only the area where cars will be in the image.  To do this I just use a simple slice on the y axis from `350` to `680`.

![cropped][]

#### 2. Heatmap

The next step is to run this cropped image through the neural network in order to get the "heatmap" of where cars are in the image.

![heatmap][]

#### 3. Scale

If you notice the <axis things> above, you'll see that the heatmap is actually much smaller since due to the network shrinking 64 to 1.  So, in order to rectify this the image needs to be scaled up to the size of the original cropped image.
 
![scaled][]

#### 4. Blur

In order to avoid false positives, I then blur the image.  This helps in two ways, firstly any single spikes in the heatmap become less noticeable and secondly blobs of many high probability areas that are close together will maintain there peak.

In this example you can see how the little spot on the left has become less prominent, but the center of the larger blobs on the right mantain there heat in the center.
 
![blurred][]

#### 5. Label

Next, I label the blobs & do a little few sanity checks.  Firstly, I reject all blobs that have a peak lower than a certain threshold.  Secondly, I reject all blobs that are smaller than a threshold.  I also have a parameter for how deep the valleys must be between mountains in order to get a seperate label.
 
![labeled][]


#### 6. Calculate & Draw boxes.

Lastly, given the labels from before I create rectangles around each of the labeled blobs and draw them onto the original cropped image.
 
![boxes_cropped][]

Or, if I shift the boxes down `350` to account for the cropping I can draw them on the original image:

![boxes][]


### Video Implementation

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (somewhat wobbly or unstable bounding boxes are ok as long as you are identifying the vehicles most of the time with minimal false positives.)
Here's a [link to my video result](./project_video.mp4)


#### 2. Describe how (and identify where in your code) you implemented some kind of filter for false positives and some method for combining overlapping bounding boxes.

In order to make use of time series images, I keep a cache of the heatmaps over time and instead of using the heatmaps directly I take a weighted average over the cached heatmaps.  This makes the assumption that cars don't teleport...  Also that they won't move really quickly.

<Insert images of time series heatmaps>*

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

One issue I've noticed is that the detector seems to be really great at detecting the _back_ of cars, but is hit and miss with the sides of cars.  I suspect this is due to the dataset coming mostly from images taken on the highway where you see a lot more of the back of cars then there sides.

Also, due to using a a time series cache, the boxes can seem to lag a few frames behind the images, this could be improved by instead of just doing a straight average by doing some sort of prediction of where the car should be and taking an average on that.

Lastly, the reason I chose to use a neural network instead of standard machine learning classifiers was two fold, firstly I wanted more experience with deep learning, but also, I am way to lazy to tune all those parameters myself.

* I apologize for the lack of images, unfortunately I had to create the writeup on a different computer than the project so I was limited to the visualizations I already had in the notebook.
