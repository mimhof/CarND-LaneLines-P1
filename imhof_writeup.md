# **Finding Lane Lines on the Road** 

### Michael Imhof
### Submitted: 01/23/20

---

**Finding Lane Lines on the Road**

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report


[//]: # (Image References)

[gray_image]: ./writeup_media/gray_image.png "Grayscale"
[yellow_mask]: ./writeup_media/yellow_mask.png "Yellow mask"
[combined_mask]: ./writeup_media/combined_mask.png "Combined mask"
[canny_output]: ./writeup_media/canny_output.png "Canny output"
[canny_with_roi]: ./writeup_media/canny_with_roi.png "Canny with roi"
[initial_lines]: ./writeup_media/initial_lines.png "Initial lines"
[kmeans_clustering]: ./writeup_media/kmeans_clustering.png "KMeans clustering"
[linear_fit]: ./writeup_media/linear_fit.png "Linear fit"
[final_lines]: ./writeup_media/final_lines.png "Final lines"

---

### Reflection

### 1. Describe your pipeline. As part of the description, explain how you modified the draw_lines() function.

My pipeline consisted of 6 steps. First, I converted the image to an HSV colorspace (Hue, Saturation, Value). 
This was done in order to highlight yellow lines against the background as under grayspace it does not stick out quite 
as well as white lines. The additional benefit of the HSV colorspace is that colors can be filtered for quite easily 
using `cv2.inRange(hsv_image, lower_threshold, upper_threshold)` where each threshold are HSV values that are meant to 
provide bounds on the color for which we are attempting to filter. To show this in action, we examine the test image
`solidYellowCurve2.jpg` below:

![A grayscale image of the yellow curve.][gray_image]

While it may seem like the yellow line (left) is plainly visible in grayscale similar to the white lines, they are
more difficult to detect in this colorspace as they just do not stick out as much as the white lines. By comparison,
when we use the method mentioned above and attempt to filter for yellow lines we can visualize it in the following
image:

![The yellow lines stick out quite a bit in HSV color space when filtered.][yellow_mask]

As can be seen, the yellow line is now abundantly visible as we have filtered for this color in HSV colorspace.
We then also filter the original image for white in HSV colorspace (using the same method described above) and take
the bitwise-or of the yellow mask with the white mask using `cv2.bitwise_or(yellow_mask, white_mask)`. This results
in the image seen below:

![The original image has been pre-filtered to detect yellow and white colors.][combined_mask]

I included in my Jupyter notebook the option of including additional Gaussian blur as used in the lessons but found
that I arrived at better results without it (not to mention the Canny algorithm already includes a Gaussian blur 
kernel).

Next we run the Canny algorithm on this image in order to identify lines in the image. To aid in the determination
of good parameters for use in both the Canny algorithm and the Hough transform, I set up an interactive tool in
the Jupyter notebook where the parameters can be tuned and the resulting image is shown. Toward that end, I used
a lower threshold of 64 and an upper threshold of 192 for the Canny algorithm. The resulting image using the image
above with these thresholds can be seen below:

![Here are the lines produced from the Canny algorithm.][canny_output]

Clearly from the figure above the desired road lines can be seen along with other road lines and objects in the 
periphery. To cut down on noise outside of the desired region of interest, we apply a trapezoidal filter to this
image where anything outside of the filter is set to 0, leaving (hopefully) only the desired road lines. The points
I chose for this trapezoid are: the bottom left corner, the bottom right corner, 47% of the width at 60% toward the
bottom of the image, and 53% of the width at 60% toward the bottom of the image. When this is applied to the image
above, it results in the following image:

![Here is the result of applying masking to the Canny output.][canny_with_roi]

We can see that only the desired road lines remain in the image and we are now ready to apply the Hough transform
to take the points identified from Canny and construct line segments. The Hough transform takes in quite a few
parameters and (similar to the Canny parameters) I found values that work by using my interactive tool in the 
Jupyter notebook. Toward that end, I ended up using the following parameters:

- phi: 1
- theta: pi/180
- vote threshold: 32
- minimum line length: 1
- maximum line gap: 200

When applying the transform with these parameters to images in the test set, the following can be seen on one of those
images:

![Initial line segments on one of the test images.][initial_lines]

From the image above we can see that the lines are clearly defined through the Hough transform. The only issue that
remains is that sometimes the line may not go to the bottom of the image and in some cases may have gaps in the line
segments. We want a cohesive line to cover the left and right lane lines. To accomplish this task, I decided to use
K-Means clustering to identify good lines and whether they belong to the left or right, and linear regression to 
determine one line that fits with the lines detected through the Hough transform (that were not rejected by clustering
outlier rejection). K-Means clustering is a type of unsupervised machine learning that (given a specification of two
clusters) assigns each point to one of the clusters. Each point in this space is a parameter set of (m, b) where m
is the slope and b is the y-intercept (in the equation `y = mx + b`). The clusters assigned based on the image above
can be seen in the figure below:

![Two clusters are produced based on left and right lines.][kmeans_clustering]

Here we have two clusters, yellow and black, that identify with left (yellow) and right (black) lines. There is
a certain amount of outlier rejection that I include based on calculating the clusters, thresholding out outliers, 
and then re-calculating the clusters. This results in reliable lines to use for constructing the overall left and
right lines. Now that the individual line segments have been identified as left and right, I apply linear regression
using the (x,y) points generated from the Hough transform. Plotting the lines produced from linear regression through
these points, we can produce the figure below demonstrating the fit produced from this approach:

![Models are chosen using linear regression that best fit the points in each cluster.][linear_fit]

We can see that the models chosen from linear regression match well with the points produced from the Hough
transform and identified through K-Means clustering. Now that we have one linear model for each of the left
and right lines, we now figure out the range of x-values over which to draw the lines on the image. This is done
by using `x = (y-b)/m` with y points chosen based on the bottom of the image and 60% toward the bottom of the image
(the same vertical range used in masking). Once these (x,y) pairs have been identified, we can then draw them
on the image using `cv2.line(image, (x1, y1), (x2, y2), color, thickness)`. Drawing these lines on the image used
above results in the final image shown below:

![The final result from the pipeline.][final_lines]

As can be seen, the lines are correctly identified and drawn from the bottom of the image up to 60% of the way
toward the bottom (40% of the image is drawn with the lines). When applied to the videos included in the project,
lines are drawn correctly through nearly every frame of the videos (there are a couple individual frames where
glitches occur).

### 2. Identify potential shortcomings with your current pipeline

One of the potential shortcomings of my pipeline is that the drawn lines are somewhat jittery when applied to
video where they correctly identify the lanes but shift around a bit. 

Additionally, the lane lines are sometimes not perfect lines, but rather are curved and perhaps lines are not the
best models for covering these lanes. 

Finally, in the challenge video, there is a transition from asphalt to concrete and the road lines are not quite
as contrasted against the pavement. This provides further challenges in thresholding the lines.

### 3. Suggest possible improvements to your pipeline

For the jittery lines in the videos, if we could include some sort of averaging across frames based on a moving
average of the parameters (m, b) then they would move more smoothly through the videos.

For the curved lane lines, we could use a polynomial fit of the points produced from the Hough transform to fit to 
the points found through the transform. We could specify a polynomial of degree n to best fit these points while
making sure not to overfit. 

For the transition to concrete, I think we need a way to tune the detection parameters against different surfaces
to try to find a way to optimally detect against any road surface.