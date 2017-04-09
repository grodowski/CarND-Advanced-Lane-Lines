## Writeup Template
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

[image1]: ./writeup_assets/undistorted.png "Undistorted"
[image2]: ./writeup_assets/distorted.png "Distorted"
[image4]: ./writeup_assets/windows.png "Warp with fitted polylines"
[image6]: ./writeup_assets/hist.png "Histogram"
[image7]: ./writeup_assets/sample_frame.png "Frame"
[image8]: ./writeup_assets/color_sobel_binary.png "Binary"
[image9]: ./writeup_assets/bare_frame.png "Frame"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

Camera calibration could be found in the second cell of the submitted Jupyter notebook. I have iterated through chessboard images provided to create inputs (image corner positions vs real world corner positions) for `cv2.calibrateCamera`. I saved the camera matrix as well as distorsion coefficients and provided a global `cal_undistort` helper function.

Distorted:

![distorted][image2]

After passing through `cal_undistort`:

![undistorted][image1]

### Pipeline (single images)

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image, found in the 3rd and 4th sections of my submission. The global `preprocess` helper combines `cal_undistort` with HLS thresholding (S-channel only) and an absolute Sobel gradient (x direction). Sample results:

Original frame:

![frame][image9]

After `preprocess`:

![preprocess][image8]


#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The perspective transform is performed by `Processor` instances as first steps of the `frame` method.

The following variables define the transformation from a trapezoid to rectangular shape.

```
warp_src = np.float32(
    [[(img_size[0] / 2) - 55, img_size[1] / 2 + 100],
    [((img_size[0] / 6) - 20), img_size[1]],
    [(img_size[0] * 5 / 6) + 20, img_size[1]],
    [(img_size[0] / 2 + 55), img_size[1] / 2 + 100]])
warp_dst = np.float32(
    [[(img_size[0] / 4), 0],
    [(img_size[0] / 4), img_size[1]],
    [(img_size[0] * 3 / 4), img_size[1]],
    [(img_size[0] * 3 / 4), 0]])
```

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

`Processor` instances find polynomial representations of each line in `__slide_window_fit`. I used the sliding window approach and started at two highest peaks of a histogram representing the lower half of the binary image.

![histogram][image6]

After finding all the pixels corresponding to the left or right lane, I used `np.polyfit` to fit two polynomials to each of the lines - the pixel-spaced and real-world spaced.

![warped_with_poly][image4]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

Curve radius and offset from the origin (left image edge) are computed and stored in each respective `Lane` instance in the `update` method, which is in turn called every time a relevant frame is processed by `Processor`.

Then, `Processor` combines data from both lanes in `center_offset` to provide a single offset value w.r.t. the image center rather than the left edge.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

`Processor` uses an inverse of the transformation matrix in `__draw_fit_unwarped` computed for warping the initial image to unwarp it and return as a final result. The function is also responsible for drawing the drivable area polygon and rendering text data (offset and curvature) on a final image.

![frame][image7]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./output_final.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.

Additional techniques:

1. Sanity check - drop lane approximation outliers. I chose to reject certain lane updates based on `abs(self.fit[0] - fit_poly2[0]) > drop_x2_thresh`. This helped avoid jitter in tricky lightning / road surface conditions.

2. Smoothing. I decided to introduce a capped collection for storing last N frames of each `Lane` and average output using these.

Problems:

- Shadows have a strong impact on the current solution and the smoothing I used works well just
  for the basic video. `harder_challenge_video.mp4` works better without the sanity checks and smoothig
  as they are too strict.
- Harder challenge video is something I could work on in the future to improve the algorithm.
- Road imperfections from `challenge_video.mp4` indicate a need for a more robust lane
  selection. `Lane.update` drops outliers by using the quadratic term only, which does not prevent the lane to jump from left to right. Doing the filtering for the second and third polynomial coefficients could help mitigate this issue.
