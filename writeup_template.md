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

[image1]: ./examples/undistort_output.png "Undistorted"
[image2]: ./test_images/test1.jpg "Road Transformed"
[image3]: ./examples/binary_combo_example.jpg "Binary Example"
[image4]: ./examples/warped_straight_lines.jpg "Warp Example"
[image5]: ./examples/color_fit_lines.jpg "Fit Visual"
[image6]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"
[image7]: ./camera_cal/calibration17.jpg "Original"
[image8]: ./camera_cal/Corrected17.jpg "Corrected"
[image9]: ./test_images/straight_lines1.jpg "Original"
[image10]: ./test_images/warped.jpg "Corrected"
[image11]: ./test_images/out_img.jpg "Combined"
[image12]: ./test_images/lane.jpg "Lane Identification"
[image13]: ./test_images/result.jpg "Result"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Camera Calibration

The camera calibration is carried out using the given set of calibration images (20 pictures of a chess-board pattern with 9x6 corners at different perspective to the horizontal plane). I ignored the images 1-5 as they were clipped in one way or the other at the edges thereby potentially affecting the calibration. The calibration is perfomed on the remaining 15 images by using opencv functions, 'findChessboardCorners' & 'calibrateCamera'. The findChessboardCorners returns the location of the corners it found and I passed them to the calibrateCamera function to get  'camera matrix' & 'distortion coefficients' variables.

To save some time, I also saved the variables to a pickle file. This way, I could re-use the pickle file when trying out different ways to tackle the lane identification problem without recalculating the calibration variables.

Another opencv function 'undistort' corrects an input image using the camera matrix and distortion coefficients variables. This is also the first in our pipeline.



### Pipeline (Test images)

#### 1. Distortion Correction

Cameras suffers from distortion and it's generally a good idea to correct for it given a set of calibration for it. Since we already cailbrated our camera and obtained the calibration parameters, we can correct for the inherent distortion in the camera. Take a look at the below two images to see a distorted(original) image and a distortion-corrected image.

![alt text][image7]
![alt text][image8]


#### 2. Perspective Transformation

Right after the distortion correction, I applied a perspective transformation. To do so, I manually selected 4 points on a test image that corresponds to a rectange in reality. I selected 4 output point coordinates to project the 4 reactange points into an actual rectangle. This transforms the image's perspective that better aid in the lane identification. 

![alt text][image9]
![alt text][image10]


#### 3. Image Processing

I tried various approaches to manipulating & extracting the image information to obtain a low noise lane feature image. After many such attempts, I arrived at a combination of different color spaces usage, thresholding, gradient characteritics & morphological operations.

Advanced Lane Finding_projectVideo.ipynb
----------------------------------------- 
This contains the code for finding lane lines in the project_video.mp4. This achivies a very good lane detetction for the given video with minimal complexity.

#### a. Color Spaces
The RGB image is transformed to HLS space and it's filtered against two masks. One to exclude green and the other to include blue. This mask was developed during the introductory lane detection project early on in the course. This seems to give a good starting point to remove most of the wanted features from the image.

#### b. Thresholding
The filtered image is then converted to a grayscale image and thresholded to yield a binary image (intensity > 50 is included).

#### c. Gradient
Using the grayscale image of the original RGB image, I calculated the gradient in the X & Y-direction and obtained magnitude of the gradient. I also calculated the direction of the gradient. After experimenting with a few combinations, I settled on the total magnitude gradient & directional gradient. Using this with the thresholded HSL image provided a robust lane binary image.

![alt text][image11]



Advanced Lane Finding_Improvements.ipynb
-----------------------------------------
When the above pipeline is tested on the challenge_video or the harder_challenge_video, the performance was very bad. In order to create a better pipeline that can detect lanes on these challenging videos, further improvments in lane detections and outlier predictions have been employed.


#### a. Color Spaces
In additional to the reading the image in the RGB format/space, I created two more version of the image in the HSL & LAB color spaces. The lane's visibility changes depending on the lane color (white vs yellow) & environmental factors like, sun-light, shadow & reflection. To extract lanes under varyinig road conditions, I primarily used S and L-channels from HSL & B-channel from the LAB space.

I created a weighted image from the S,L & B-channels with the L-channel weighing twice S & B-channels. This weighted image is scaled back to the 0-255 range later.

#### b. Morphological Operations
This weighted image is then processed with an opening operation using a horizontal element (1x30). The result of this operation is subtracted from the weighted image obtained in the prior section to arrive at a grayscale image that highlights the lane marking.

#### c. Thresholding
I thresholded the HSL image to filter out green but include blue information from the image. This filtered image is then converted to a grayscale image and thresholded to yield a binary image.

#### d. Gradient
Using the grayscale image obtained from the morphological operations above, we find the gradient in the X & Y-direction and obtain magnitude of the gradient. We also calculate the direction of the gradient. This directional graident is further enhanced by opening on 5x5 kernel.

Finally, I arrived at a binary thresholded image using the magnitude of the gradient, directional gradient & the thresholded HSL image.


![alt text][image11]



#### 4. Lane identification & fitting
With the thresholded image in hand highlighting on lane pixels, I used the bottom half of the image and looked at the historgram to find peaks that can be reprsentative of the lane. Dividing the image vertically into two halves, I picked the highhest peak in each half as the origin of the left and right lane respectively.

Since lanes tend to maintain constant seperation between them and also change curvature relatively slowly (exceptions are acknowledged), we search a small window(rectangle) around the lane origin to find the lane points as we make our way verically in the image while sliding the window along the lane center. This gived us the left and right lane for this image.

![alt text][image12]

Once we identified the lane for an image, for subsequent images, we can forego the histogram/sliding window search and just look for the lane with in a margin of the original lane. This simplication is due to a realistic assumption that lane changes cannot be drastic between frame rates consideration an average frame rate of the camera.

Improvements to the lane identifiation & searching has been done by weighing in past frame's lane position with anew frame and deciding whether the new frame lanes are real or an outlier based on pre-selected error thresholds. If it is detrmined to be a bad frame, I re-did the historgram/sliding window search to reestablish the lanes.



#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The radius of curvature and center offset are calculated per frame and drawn on the image. This adds some insight into road conditions and a quick check aginst the pipeline performance.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Below image shows the lanes mapped back to the original image (after converting back from the perspective image).

![alt text][image13]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output_project_submission.mp4)

Here's a [link to my project video result without any added improvements](https://youtu.be/Vmh-w_hI2Fk)
Here's a [link to my project video result with added improvements](https://youtu.be/qmm8nCfrxjs)



---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The improved pipeline begins tracking the lanes well but goes off later. It does a good job of reacquiring the lanes but improvements need to be made to keep the tracking go bad. To that end, I started using a weighted average for adding new lanes that helps in reducing drastic changes between each frame. This still ends refinement. More importantly, outlier elimination also needs some work as the bigger the outlier the higher of an effect it cna have on the weighted average lane positions. A better image manipulation to extract the lanes under different road conditions and reflections will help towards that end.


Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
