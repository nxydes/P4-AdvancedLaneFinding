## Advanced Lane Finding
### Project 4
### Nick Xydes

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

[image1]: ./output_images/calibration9.jpg "Raw Calibration"
[image2]: ./output_images/annotated.jpg "Annotated Image"
[image3]: ./output_images/undistorted.jpg "Undistorted Image"
[image4]: ./output_images/undistorted_car.jpg "Undistorted Vehicle Example Image"
[image5]: ./output_images/thresholded.jpg "Thresholded undistorted example image"
[image6]: ./output_images/annotated_flattened_test_images/test6.jpg "Lane Lines Detected"
[image7]: ./output_images/annotated_test_images/test6.jpg "Projected onto lane"

[video1]: ./project_video.mp4 "Video"

---
## Project Results

Please note that my submission is divided into two parts. The first is developing the pipeline for individual test images and is completed in the Project.ipynb file. The second is modifying this pipeline to work with videos and do filtering between frames. This is handled in Video_architecture.ipynb


### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first two code cells of the Project.ipynb notebook file, the output image is contained in output_images/annotated.jpg.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

Then, I use the cv2 function findChessBoardCorners on each image with the expected chessboard size. This gets me a return matrix of their locations if all of the corners are found. I can then use this to draw the chessboardcorners onto the image as you see in the example image below. 

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained the following result. 

Raw calibration image:
![alt text][image1]

Annotated with the found chessboard corners:
![alt text][image2]

Raw image that has had the distortion removed:
![alt text][image3]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

The first step in the pipeline is to undistort the raw image coming from the cars camera. This is simply done as was described and shown in the calibration step with one line of code at the top of the fourth code cell. The results are as shown:
![alt text][image4]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

This was one of the most difficult parts, not because the code was difficult but because tuning it has been challenging. I knew I would need to combine many types of thresholds to get just the lines to show up in my end result. In the process of attempting various things I ended up trying many different types of thresholds. These included using the Red channel of the RGB color scheme, using combinations of each of the H,L and S color channels in the HLS color scheme and using Sobel transforms along the X, the Y and both axes. I tried combining these different channels in `AND` or `OR` methods.

Ultimately, I ended up using the H and S channels in and AND relationship combined with the Sobel along the X axis. This got me good results for the project video, even in poor lighting. However, it ended up with the worst of results in the challenge video, it tended to grab onto the cracks in the road and not the yellow side marking. This was rather annoying but I focused on the project video first.

An example of the thresholds I ended up using is below:

![alt text][image5]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.
#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The code for my perspective transform includes a function called `warp()`, which appears in the 3rd code cell of the IPython Project notebook.  The `warp()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points. 

To get the source corners, I actually measured the pixel locations of the lane in one of the images where it passes the bottom of the images and made sure this was symettric around the center of the image. Then I found the lane boundaries using the same methodology but at the same height in the image.

For the destination points, I went with the same methodology as the examples. I used the same numbers here because it allows the lane to be roughly in the right spot in the image and take up the whole vertical space of the image.

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 205, 720      | 320, 720      | 
| 1110, 720     | 960, 720      |
| 600, 448      | 320, 0        |
| 680, 448      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

Then came the difficult task of finding the lanes in the image. To do this, I followed the sliding window search method of the tutorials. I started with taking a histogram of the bottom half of the transformed image. This allows me to search for the highest threshold beginning point on the left and right side of the image (for the left and right lanes).

Next, I looked at a window around the estimated beginning of the lane. This window margin was set to roughly 150 pixels in the x and the height was configured for a total of 9 windows. For each window, we start at the previous best pixel location (or the histogram starting location) and we search for all pixels that are nonzero within this window and append to a list. As we go up the image, we recenter the next image search on the mean of all the pixels in the previous window.

Then, once we have all the indexes for our estimated lane, we can do a numpy polyfit of a 2nd degree polynomial on the lane markings. This is our estimate for the lane markings! However, we want to display this, so we can take for each y pixel, plug it into the second order equation and pop out its x pixel location, this allows us to draw the lane marking on the image.

You can see an example of this pipeline below:

![alt text][image6]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

This was one of the simplest tasks and was completed in the 12th and 13th code snippest in the Project Notebook. To do this, I found the offset to the center by calculating where the lane lines intersected the bottom of the image and calculated the offset of their mean from the center of the bottom of the image, then converted this to world frame in meters.

The curvature calculation is mildly more difficult. I fit new polynomials in the world space and then calculate the curvature using the first and second coefficients of the polynomial (from taking the derivative of the curve). There is a simple formula for this as given [in this tutorial](https://www.intmath.com/applications-differentiation/8-radius-curvature.php)

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in the 14th code block in the Project notebook in the function `draw_on_original`.  This is accomplished by recasting the x,y coordinates of the lane markings into the format that cv2 wants, then using the cv2 function `fillPoly` to fill up the space in between with green. Finally, I warp this information back onto the original image space using cv2 `warpPerspective` and then combine the result with the original undistorted image. 

Here is an example of my result on a test image:

![alt text][image7]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./output_images/project_video_output.mp4)

The code for the video pipeline is contained in Video_architecture.ipynb.

I made major changes to the pipeline to get the video to work consistently. I obviously started with the image pipeline and just doing it consecutively but this time I really wanted to follow the directions and do averaging over multiple frames and be more intelligent with lane selection. 

To do this, the majority of my changes involved modifying many of the formulas to take polynomials of the lane lines instead of the x,y coordinates of the lane pixels. This allowed me to store the array of polynomials in an array of arrays and average the last ~5 frames to get a better estimate of the lane. This seriously helped when the video passes through the challenging bridges and shadowy regions. 

I made a few decisions to make the selection more intelligent. First, if I had a good estimate from the previous frame of the lane line, I would start there and do a sliding window search instead of doing the histogram initialization. Second, I would throw out detections and start over if I met a couple of conditions. The first condition being if the left and right detected lane curvatures were more than 2x different. The second condition being if the cooeficients of the detected lane were too different from the lanes current best fit. 

For simplicity, if either of these conditions were met both lane lines would restart with histogram filters until both found good lines again. 

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

I discussed many of the problems and approaches in the above sections. I see this project as having 2 major parts, the color transformation/thresholding and the lane detections.

My color pipeline seems to work only on the example project video. I believe this to be due to me tuning the color thresholds on this specific road and my results not being transmissable to other situations. I noticed others in the slack channels had used other types of color thresholding or were more specific to only look for white and yellow. I would like to try this in the future, as I noticed the challenge video tends to grab onto the dark crack in the ground and not the yellow lane. 

Aside from the color pipeline, I really want to improve the lane parameters. First, I think it'd be cool to specify more parameters on the lanes, such as, they must be x+/- distance apart and they must maintain this throughout the image. I think moving the sliding windows laterally in unison would help a lot, instead of independently. Furthermore, I would modify it to use the other lane with some offset when one lane loses its detection and needs to start searching again.

Overall, I'm pleased with my architecture but should I continue this for a career there are many portions that could be improved. I also think machine learning might do a better job at accomplishing this task.
