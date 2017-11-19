## Advanced Lane Finding Project

---

**Goals of this project**

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./output_images/camera_calibration.png "Camera Calibration"
[image2]: ./output_images/undistort_output.png "Undistorted"
[image3]: ./output_images/binarization.png "Binarization"
[image4]: ./output_images/perspective_transform_straight.png "PT Straight"
[image5]: ./output_images/perspective_transform_curve.png "PT Curve"
[image6]: ./output_images/sliding_window_search.png "Fit Visual"
[image7]: ./output_images/curvature_and_offset.png "Stats"
[video1]: ./output_videos/project_video.mp4 "Video"


---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "./lane_finding.ipynb". 

The following steps are followed:
* Choose a particular chessboard (9 x 6) and took a lot of picutres of it (in this case, I just used the ones provided)
* Create grid points to where I expect each chess cell corner is
* For each of the chessboard image:
    - covert to grayscale image then find its corners
    - adding the pair of points where matches the image space chessboard corner with the corners in chessboard space

Below is an example of the original image comparing to undistored image: 

![alt text][image1]

### Pipeline (single images)

#### 1. Undistort Image

With the point pairs from the calibration process, I got the calibration matrices, which can then be used to undistort any images, using the opencv3's `undistort` function. And below is a comparison of the original and undistorted images on a chessboard. 

![alt text][image2]

#### 2. Image Binarization

In code cell `[3]` in the ipython notebook, I used a combination of color and gradient thresholds to generate a binary image. I experimented with both the HLS and HSV channels along with the sobelx computuations on the image. In addition, I played with the threshold parameters to try to get some binary image that looks good to human eyes. Here is the combination I've eventually chosen:
* Sobelx - 20 to 100 (discenable to verticle edges)
* saturation channel - 200 to 250 (good for rich color areas)
* value channel - 220 to 255 (relatively high reflective surfaces)

Below is an example of the binarization of one of the example images:

![alt text][image3]

#### 3. Perspective Transformation

In the code cell `[4]`, I implemented the perspective transformation. In this step, the key thing is to provide a good pair of source and destination points for the transform to be computed correctly. To aid this process of finding a good pair of sorece and destination points, I additionally plot the points I hand picked onto the original and warpped images. See the example below. 

After carefully hand-tuning and visualizing the results, I finally picked the following points for computing the transforms:
```python
src = np.float32([(565, 470), (720, 470), (1050, 670), (270, 670)])
dst = np.float32([(270, 100), (1050, 100), (1050, 720), (270, 720)])
```

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto multiple test images and their warped counterparts to verify that the lines appear parallel in the warped images. Below are two examples of the comparison, for for straight lines and the other for curved lines.

![alt text][image4]
![alt text][image5]

#### 4. Fitting a Polynomial

In code cell `[6]`, I defined a function to extract points that apprear to be from the lane lines. It is a hard judgement for a machine, so using a historgram over the lower half of an image is a good starting point. Using lowever half is because it is closet to the camera which is the least warpped and have the higest resolutions on the lines. 
After that I follow the below procedures to forward search more points on the image:
* Set up windows of search area from the bottom of the image, one search window per side of lanes. Staring from the two highest concentration of points detected from the historgram.
* sliding upward by some fixed amount of distance, and collect the points that are within this current windows
* check which side has the higher concentration of points in the windows,
* compute the mean in the x aixs of the all thse points, then the difference of the new mean and the previous center as the offset distance to move the searching window in the x direction when it searches forward in the y direction. 
* Also, add all the collected points into the big collection of points

With all these points collected, use the numpy `np.polyfit` to fit them in to 2nd order polynominal lines, one for each side of the lane. See below for an example:

![alt text][image6]

#### 5. Computing Curvature and Center Offset.

In the code cell `11`, I wrote functions to do the computations. 

For the curvature, basically, I just used the formula provided in the course to do the computation of curvature based on the fitted polynomial functions. And I used `3.7/700` meters per pixel for the `x` dimension and `30/700` meters per pixel for the `y` dimension. Then I average up the left curvature and the right curvature to be the curvature of the lane.

For the center offset. I assumed that the camera is mounted to the center of the car. So the center of the image is the center of the car. I compute the distance between the center of the image and the center of the detected lane region's base (i.e. closet to the car). And if the distance is negative, then the car is to the right side of the lane center; otherwise, it is on the left side of the lane center. 

See example for the computation in one frame shown in the image below in the next section.

#### 6. Unwarpping image

In code cell `[10]`, I defined a function to unwarp the top-down image. This is easily achieved by using the perspective transform function again, except this time, I need to provide the original destination points as the new source points and the original source points as the new destination points. This give me the inverse of the original transform. 

Below is an example of the overlaying the detected lane region back to the original image. 

![alt text][image7]

---

### Pipeline (video)

#### 1. Testing the pipeline
The pipeline for single images described above is applied to the `project_video.mp`, and below is the result:

Here's a [link to my video result](./output_videos/project_video.mp4)

---

### Discussion

#### 1. Issues and Systematic Approach for Improvement

For challenge videos, cases where the pavement coloration is not uniform, and the shadow under the bridge is too high, or when the curve is too much, the pipeline doesn't perform well. There are examples `output_videos/challenge_video.mp4` and `output_videos/harder_challenge_video.mp4`. There are a few ways to debug the issue and see why the pipeline is not performing well under these situations:
* first, one may just isolate those frames using the `subclip` of the video tool to capture only those frames
* second, instead of apply the entire pipeline to them, just apply each step to separately to those problematic frames
* third, fine-tune parameters for the thresholds or choose different combinations for binarizations to get improved images. 
* apply to the video, and see the results, then iterate from the first step


#### 2. Smoothing out Fitted Parameters
However, just from intuition. One big improvement that can be made is to use the history of the fit parameters to forward tuning the area of lane line search in the future frames. In addition, since the lane lines themselves don't change too much, some moving averaging or kalman filtering of the lane line parameters could also help smooth out the fitted lines from frame to frame. Oh well, if only if I have enough time...
