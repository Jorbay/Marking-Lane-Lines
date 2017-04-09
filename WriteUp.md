#Writeup for Project 4 of Udacity's Self Driving Car Engineer Nanodegree

---

TL/DR: I made a  pipeline that takes in videos of a car driving in a single lane! The pipeline outputs the same video with the currently occupied lane line marks and the current radius of curvature pertaining to the lane. The software can all be found in the jupyter notebook file "LaneLines_script"

**Advanced Lane Finding Project**

[image1]: ./WriteUp_pics/distorted.jpg "Distorted"
[image2]: ./WriteUp_pics/undistorted.jpg "Undistorted"
[image3]: ./WriteUp_pics/binary.jpg "Binary"
[image4]: ./WriteUp_pics/warped.jpg "Warped"
[image5]: ./WriteUp_pics/binary_lanes.jpg "Marked Lines"
[image6]: ./WriteUp_pics/finished_frame.jpg "Finished Product"

Goals for this project:
* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.


Write up Rubric points:

**Camera Calibration and Undistortion**
#Rubric Criteria 2 and 3
	As with all lens cameras, the camera we used for this project creates slightly distorted pictures. Luckily, the same person who filmed the car driving also took pictures of chessboards at several orientations. The corners of every square of a chessboard are regularly spaced out, so I can assume that they should appear as a regular grid. However, because of camera distortion, the chessboard does not appear exactly like a regular grid. I solved this problem in the first cell of the jupyter notebook file "LaneLines_script".

	I first create a set of points called objpoints, which maps out where I expect to find the chessboard's square's corners, and imgpoints, which maps out where the corners are actually found. For both sets of points, I map them out in 3D space and estimate their position in depth as z=0.  I then use the open cv function calibrateCamera, which takes in both sets of points and returns the camera matrix (a matrix that contains the focal lengths and optical centers of the camera; multiplying any pixel vector by the camera matrix will return the "undistorted" position of the pixel in reality). To undistort, I used the simpler method of using opencv's undistort method, which takes in the distorted picture and the camera matrix.

	To see an example, look below:

![alt text][image1]

![alt text][image2]

**Pipeline**
#Rubric Criteria 4
	So, after undistortion, we make a few changes to the image to get a binary image that ideally only maps out the lane lines. First, in the second cell of LaneLines_script, I convert the picture from a BGR color space to an HSV color space on line 13. This is done to more easily obtain masks of only the yellow and white pixels in the picture (the colors of the lane lines I am searching for). From lines 15 to 29, I create masks for each color and use opencv's bitwise_and function to obtain all pixels that pertain to yellow and white in the image. The colors were obtained by searching for pixels that fell within the relevant HSV ranges. Then, I use opencv's bitwise_or function to combine the yellow and white pictures, which leaves we with a single picture with only yellow and white pixels.

	Now, I use opencv's cvtColor function to convert the yellow and white picture  to grayscale (the same function was used ot convert the picture to HSV). This is followed by a Sobel operator, which I tailor to obtain slightly verticall lines. The results of this operator are  scaled from 255 to 0, and then the values are binaried so that all values greater than 30 go to 1 and all values less than 30 go to 0. An example of a binary picture is seen below. 

![alt text][image3]

#Rubric Criteria 5
	Now, I transformed the binary picture from a picture of the road from the car's perspective to a bird's eye view of the road. This is done in the fourth cell of LaneLines_script, in lines 44 to 52. First, I record the location of the corners of the road in the perspective of the car, which forms a trapezoid that is wider toward the bottom and stretches out toward the horizon in the middle of the picture. I then record the desired translation of the road's corners in a bird's eye picture, which would be a regular rectangle that spans the view. I use opencv's getPerspectiveTransform function to get a matrix that, when multiplied by point coordinates in the original perspective, transforms them to their respective locations in the bird's eye perspective. I then use openCV's warpPerspective function to transform the binary picture to a bird's eye binary perspective, which can be seen below.

![alt text][image4]

#Rubric Criteria 6
	Now, we must trace out the polynomial that will be fit with the lane pixels to map out the lane. This is done in the fourth cell of LaneLines_script from line 63 to 156. This method, much like the transformation section, is highly based off of examples provided by Udacity's Self Driving Car Nanodegree classes. 
	
	We start by obtaining a histogram of the horizontal positions of pixels in the bottom half of the current binary picture. The horizontal position with the most amount of pixels on the left half of the picture is considered the starting location of the left  lane, whereas the horizontal position with the most amount of  pixels on the right half of the picture is considred the starting location of the right lane. Then, we create a small window around both beginning lane locations of width 200 and height of about 70. We add all the points within the window to a list of points called "left_lane_inds" and "right_lane_inds" (lane indicators). If there are more than 50 pixels in the current window, we calculate the mean of the horizontal position of the pixels and use it as a center  for the next window. We then build new windows above the previous windows and repeat the process until we travel vertically across the entire picture, gathering all pixels that pertain to either the left or right lane. I designed the algorithm to use 9 sets of windows (so 9 windows per lane) for every picture, allowing the windows to move if they are following curved lanes. 

	With the collected pixels, I use numpy's polyfit function to fit a quadratic function (so a function with a degree of two) to the set of pixels pertaining to each lane. If no pixels were collected, which happens rarely for the project_video, the quadratic function of the previously recorded frame is used. Each quadratic function is recorded as a vector of three constants (if Ax^2 + Bx + C is every possible quadratic function, the vector is <A,B,C>). If the normal difference between this vector and the previously recorded vector is greater than some threshold (in our case, 150), we consider  the change too great and just use the quadratic function of the previously recorded frame.

	Now with a quadratic function for the left and right lane, we go through all values of y in the picture and compute their x values following the quadratic functions. We collect the computed x values into sets of points called left_fitx and right_fitx, which we will use to trace out our lane lines. To show the information we currently have, I have graphed an example below that plots the left_fitx(blue) and right_fitx (red) with their respective y values into the binary picture.

![alt text][image5]


#Rubric Criteria 7
	
	Now, we can use the quadratic function of each lane line to compute both curvature of the lanes and the car's displacement relative to the center of the lane. The code for this can be found in the fourth cell of LaneLines_script, in lines 192 to 230. 

	We start by recording that in our transformed picture, the width of the road is about 3.7 meters and the length of the road from the car to the horizon is about 30 meters (source: Udacity Self Driving Nanodegree). With this information, we can derive a new quadratic function that reflects real distances by multiplying x coordinates in the set of lane points by 3.7/900 (width of road in meters divided by width of road in pixels) and y coordinates in the set of lane points by 30/720 (length of road in meters divided by length of road in pixels). We use these new points with numpy's polyfit again to make a new quadratic formula.

 	We can derive the radius of curvature (ROC) with the following equation: (1+y'^2)^(3/2)/y''. y' is the first derivative of the new quadratic formula. y'' is the second derivative of  the new quadratic formula.

	If we use the ROC equation on each lane line's quadratic equation at the point closest to the car (the greatest y value), we obtain two values for the ROC. I used the mean of these values as the ROC for the whole lane.

	Now, for the displacement of the car from the lane's center, all we need is the location of each lane's closest pixel to the car (the bottom of the screen). We obtain that from right_fitx and left_fitx and will call them L and R. We can find the position of the center  of the lane in the picture with (L+R)/2 = center. We can assume the center of the car is always at the center of the picture (640). This is a reasonable assumption if the camera is centered on the car and never moved. To find the displacement, we can then use abs(640-center)*3.2/900. The 3.2/900 constant converts the distance from pixels to meters. If 640-center is negative, we know that the displacement is to the left of the lane center. If 640-center is positive, we know the displacement is to the right of the lane center.


#Rubric Criteria 8
	With our lane lines, we now want to unwarp back to the perspective of the car. This is all done in the fourth cell of LaneLines_script in lines 234 to 265.  We can do so by using openCV's getPerspectiveTransform again, but now switching the transform's direction so we have the inverse of the previous transform matrix. We use warpPerspective with this inverse matrix to create an unwarped image of the lane lines. We then add it to the color picture of the road by using cv2's addWeighted function with the original and lane picture.

	We add the information on displacement and curvature by using cv2's putText function.	

	An example of this is seen below:

![alt text][image6]

#Rubric Criteria 9
	To see an example of the final labeled project  video, please view ./project_video_output.mp4

**Discussion**
	So the described pipeline works well on the project_video, but  performs rather poorly on the challenge videos. First, the pipeline seems to fail when the image  is saturated by light. I suspect that this could be solved if binary pictures changed the values of white and yellow that they detected depending on the average saturation value. The higher  the average saturation value, the more strict  the pipeline should consider the definitions of yellow and white. 

	I've also noticed that the project_video seems to hesitate a bit with different colors of asphalt. I believe a similar solution as the previous may help solve this problem. If the car is constantly computing the "average" color of the road, it can adjust what it considers the colors of the lane lines to make sure that what it thinks are lane lines never takes up a large portion of the road. 

	Another improvement the pipeline could use is its accuracy in measuring curavture. If I use a running average for curvature so that the computed curvature is always weighted by the previous curvature, I believe  that should be  of great help.

	Please reach out to me at jao2154 at columbia dot edu if you have any questions.  	
