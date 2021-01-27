## Writeup

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./test_images/test1.jpg "Original image"
[image2]: ./output_images/d_test1.jpg "Undistorted"
[image3]: ./output_images/b_test1.jpg "Binary Example"
[image4]: ./output_images/w_test1.jpg "Warp Example"
[image5]: ./output_images/p_test1.jpg "Fit Visual"
[image6]: ./output_images/o_test1.jpg "Output"
[video1]: ./output_videos/project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it! Please have a look at the submitted [notebook](CarND-Advanced-Lane_Lines.ipynb) and also the [HTML-export] (CarND-Advanced-Lane_Lines.html
Note: All processed test images can be found in the folder output_images. A prefix was added to the original names depending on the processing steps that had been executed:
- Undistort: d_
- Binary thresholded: b_
- Warped: w_
- Line fit: p_
- Original image with lane: o:

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "./CarND-Advanced-Lane_Lines.ipynb".

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction first to the calibration and then to the test images using the `cv2.undistort()` function. The results can be seen in the notebook where for each test image the original and undistorted image are plotted side by side. The coefficients are stored in global variable for use in further image and video processing.

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

![alt text][image1]

With the distortion coefficients I undistorted all test images. As an example see the original test1.jpg above which results in:

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The code has been placed in a function `thresholded_binary()` (fifth code cell in the IPython notebook) to be easily reusable in further steps. The thresholded binary image is a superposition of two different images, one produced by converting the original image to HLS color space and passing the s-channel to a threshold filter cutting of values above and below certain values which are given as input parameters. The other part is the result of the application of a sobel-x filter to the grayscale of the original image also passed to a similar threshold filter but with different cut-off values.
Here's an example of my output for this step:

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform can be found in the function `warp()` (seventh code cell in the IPython notebook).  The `warp()` function takes as inputs an image (`img`), the source (`src`) and destination (`dst`) points are hardcoded in this function in the following manner:

```python
    src = np.float32(
        [[(img_size_x / 2) - 60, img_size_y / 2 + 100],
         [((img_size_x / 6) - 20), img_size_y],
         [(img_size_x * 5 / 6) + 60, img_size_y],
         [(img_size_x / 2 + 65), img_size_y / 2 + 100]])
    dst = np.float32(
        [[(img_size_x / 4), 0],
         [(img_size_x / 4), img_size_y],
         [(img_size_x * 3 / 4), img_size_y],
         [(img_size_x * 3 / 4), 0]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 580, 460      | 320, 0        | 
| 193, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 705, 460      | 960, 0        |

The function calculated and returns the forward transformation matrix `M` and the inverse Matrix `Minv` which is used for the reverse (backward) transformation.
 
I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto all test images and its warped counterpart to verify that the lines appear parallel in the warped image. For adjustment of the source parameter I used the two test images showing a straight driving situation.
This is what it looks like for one of the example images:

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The complete code can be found in several functions in the nineth code cell of the IPython notebook. Function `find_lane_pixels()` contains the functionality described in chapter "Locate the Lane Lines" and "Implement Sliding Windows". First the start of the lines is detected via histogram of the bottom half of the binary thresholded warped image which is given as an input parameter to the function. Then sliding windows are used to track the line to the top of the window. This means that the image is covered with a number of windows with configurable size that are stacked one on top of the other. Subsequently from the first window at the bottom each window is shifted horizontally to center around the detected lane line pixels.
An alternative algorithm to `find_lane_pixels()` is implemented in the function `search_around_polynomial()`. In the binary thresholded warped image which is given as an input the lines are searched in a certain margin of the polynomial of the last frame. The coefficients of the last polynomial have to be supplied as input parameters.
In the function `fit_polynomial()` which takes the line detection points for both lines as an input, the polynomials are fitted to these. The polynomial lines and the real world radius of curvature are calulated and returned next to the coefficients of the polynomials.
Here is the output of one of the images as an example:

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The radius of curvature is also calculated in the function `fit_polynomial()` with the help of the formulas given in the chapter "Measuring Curvature" and transfered to real-life dimensions. This calculation is performed for the bottom point of the window which is closest to the car.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in code cell ten and also in the procedure `process_image()` which is used for video processing.

Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./videos_output/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The processing of the images was quite straight-forward thanks to the good and detailed description of the course. The real difficulties occured in the video processing. Even in the project_video.mp4 the algorithm had problems detection lines when the lighting situation become difficult. This of course became worse in the challenge.mp4 not to mention the harder_challenge.mp4. Besides this the radius of curvature fluctuated in several sections of the video and showed substantially different values for both lines. The idea of storing value from previous frames and dropping low-quality frames seems quite good but its success is hard to evaluate because the videos improve gradually which is hard to see. My version of this inter-frame improvement is surely just the start of improvements that can be realized by this means.
To make the pipeline more robust one could start at optimizing the detection parameters using images of critical situations of all the videos. The detection of line points and the fitting of the polynomials seems to be a crucial step. Even in the test images sometimes right and left polynomials deviate and this of course leads to different radius of curvature. A refinement or even a different algorithm could solve this problem. Another possibility would be to do some post-processing and determine the quality of the line fit. Usually the intermittant line is harder to interpolate, so in most of the cases the solid line could be used as reference for the curvature and the intermittant line re-fitted based on the coefficients of the solid line.


