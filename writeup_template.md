# **Finding Lane Lines on the Road** 

## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file. But feel free to use some other method and submit a pdf if you prefer.

---

**Finding Lane Lines on the Road**

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report


[//]: # (Image References)

[image1]: ./examples/grayscale.jpg "Grayscale"

---

### Reflection

### 1. Describe your pipeline. As part of the description, explain how you modified the draw_lines() function.
initial_image = np.copy(image)
    ysize, xsize = image.shape[0:2]
    image = grayscale(image)
    image = grayscale_select(image, 200)
    image = gaussian_blur(image, 3)
    edges = canny(image, 25, 100)
    hough = hough_lines(edges, 1, np.pi/180, 40, 5, 2)
    hough = region_of_interest(hough, np.array([[[0, ysize], [xsize//2, ysize//2], [xsize, ysize]]]))
    image = weighted_img(hough, initial_image)
    return image

My pipeline consisted of 6 steps:

1. Convert the image to grayscale
2. Filter out dim pixels below `brightness_threshold=200`
3. Blur the image using a 3x3 Gaussian filter
4. Run Canny edge detection with `low_threshold=25, high_threshold=100`
5. Run Hough line detection with `rho=1, theta=1 degree, threshold=40, min_line_length=5, max_line_gap=2`. I initially used a longer `min_line_length` but found that the "dotted" lines that separate lanes are too foreshortened to be reliably detected at higher values.
6. Crop the region of interest to filter out irrelevant lines detected above the road. I simply used the bottom quarter triangle of the image as the region.

All parameters were ultimately found and tuned with trial and error. In order to draw a single line on the left and right lanes, I modified the draw_lines() function by grouping detected lines into `left`, `center` and `right` buckets based on the slope of the line. Lines close to vertical became `center` lines and lines close to horizontal were thrown out because it is unlikely that they represent a lane detection. `left` and `right` lines were classified by negative and positive slope respectively. I then ran a linear regression on the points in each bucket to compute the "average" line. I selected appropriate endpoints by getting the bounding box of the points, and truncated the lines before they could intersect by computing their intersection point. Finally, I drew the lines with `cv2.line` with `thickness=10` to make a nice clear visual signal.

A result of the pipeline:
![A result of the pipeline][test_images_output/soldWhiteRight.jpg]


### 2. Identify potential shortcomings with your current pipeline

##### Shortcoming 1
When there are other linear markings or artifacts on the road, the `draw_lines` subroutine might mix the detected lines into either the right or left lane buckets, so that when the linear regression is performed the points used belong to wildly divergent lines. This would lead to a detected "lane" that is way off.

##### Shortcoming 2
When the ambient light is very dim, for example on a cloudy day or in the shade, the lane markings may not exceed the brightness threshold, leading them to be eliminated during the first stage of the pipeline.


### 3. Suggest possible improvements to your pipeline

One way to mitigate Shortcoming 1 would be to cluster the lines within each bucket and discard lines that fall outside the most popular cluster. Assuming that the true lane is typically represented in the most popular cluster, this would eliminate irrelevant lines. The clustering threshold could be tuned to achieve good performance.

One way to mitigate Shortcoming 2 would be to use a dynamic brightness threshold: first measure the average brightness across the image and then set the brightness thresold to filters out some fraction (e.g. 90%) of pixels, leaving only the 10% brightest pixels for edge and line detection.
