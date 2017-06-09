## Project: Search and Sample Return
### Writeup Template: You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---


**The goals / steps of this project are the following:**  

**Training / Calibration**  

* Download the simulator and take data in "Training Mode"
* Test out the functions in the Jupyter Notebook provided
* Add functions to detect obstacles and samples of interest (golden rocks)
* Fill in the `process_image()` function with the appropriate image processing steps (perspective transform, color threshold etc.) to get from raw images to a map.  The `output_image` you create in this step should demonstrate that your mapping pipeline works.
* Use `moviepy` to process the images in your saved dataset with the `process_image()` function.  Include the video you produce as part of your submission.

**Autonomous Navigation / Mapping**

* Fill in the `perception_step()` function within the `perception.py` script with the appropriate image processing functions to create a map and update `Rover()` data (similar to what you did with `process_image()` in the notebook). 
* Fill in the `decision_step()` function within the `decision.py` script with conditional statements that take into consideration the outputs of the `perception_step()` in deciding how to issue throttle, brake and steering commands. 
* Iterate on your perception and decision function until your rover does a reasonable (need to define metric) job of navigating and mapping.  

[//]: # (Image References)

[image1]: ./misc/rover_image.jpg
[image2]: ./calibration_images/example_grid1.jpg
[image3]: ./calibration_images/example_rock1.jpg 
[image4]: ./output/test_mapping.png 
[image6]: ./output/image6.png
[image7]: ./output/image7.png 
[image8]: ./output/image8.png 
[image9]: ./output/image9.png 
[image10]: ./output/image10.png 
[image11]: ./output/image11.png 
[image12]: ./output/image12.png 
[image13]: ./output/image13.png 
[image14]: ./output/image14.png 

## [Rubric](https://review.udacity.com/#!/rubrics/916/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

You're reading it!

### Notebook Analysis
#### 1. Run the functions provided in the notebook on test images (first with the test data provided, next on data you have recorded). Add/modify functions to allow for color selection of obstacles and rock samples. 

## (Done.  Check file called Rover_Project_Test_Notebook.ipynb located in the code folder)
Here is an example of how to include an image in your writeup.

![alt text][image1]

#### 1. Populate the `process_image()` function with the appropriate analysis steps to map pixels identifying navigable terrain, obstacles and rock samples into a worldmap.  

Done.  See Code below.  This code can also be found in the file titled Rover_Project_Test_Notebook.ipynb

    def process_image(img):
    # Example of how to use the Databucket() object defined above
    # to print the current x, y and yaw values 
    # print(data.xpos[data.count], data.ypos[data.count], data.yaw[data.count])

    # TODO: 
    
    # 1) Define source and destination points for perspective transform
    
    # Define calibration box in source (actual) and destination (desired) coordinates
    # These source and destination points are defined to warp the image
    # to a grid where each 10x10 pixel square represents 1 square meter
    dst_size = 5 
    # Set a bottom offset to account for the fact that the bottom of the image 
    # is not the position of the rover but a bit in front of it
    # this is just a rough guess, feel free to change it!
    bottom_offset = 6
    source = np.float32([[14, 140], [301 ,140],[200, 96], [118, 96]])
    destination = np.float32([[image.shape[1]/2 - dst_size, image.shape[0] - bottom_offset],
                      [image.shape[1]/2 + dst_size, image.shape[0] - bottom_offset],
                      [image.shape[1]/2 + dst_size, image.shape[0] - 2*dst_size - bottom_offset], 
                      [image.shape[1]/2 - dst_size, image.shape[0] - 2*dst_size - bottom_offset],
                      ])
    
    # 2) Apply perspective transform
    warped = perspect_transform(img, source, destination)
    
    # 3) Apply color threshold to identify navigable terrain/obstacles/rock samples
    threshold_terrain = color_thresh(warped)
    threshold_obstacles = obstacles(warped)
    threshold_rock = rocks(warped)
    
    # 4) Convert thresholded image pixel values to rover-centric coords
    terrain_xrcc, terrain_yrcc = rover_coords(threshold_terrain)
    obstacles_xrcc, obstacles_yrcc = rover_coords(threshold_obstacles)
    rock_xrcc, rock_yrcc = rover_coords(threshold_rock)
      
    # 5) Convert rover-centric pixel values to world coords
    xpos = data.xpos[data.count]
    ypos = data.ypos[data.count]
    yaw = data.yaw[data.count]
    world_size = 200
    scale = 10
    obstacle_x_world, obstacle_y_world = pix_to_world(obstacles_xrcc, obstacles_yrcc, xpos, ypos, yaw, world_size, scale)
    rock_x_world, rock_y_world = pix_to_world(rock_xrcc, rock_yrcc, xpos, ypos, yaw, world_size, scale)
    navigable_x_world, navigable_y_world = pix_to_world(terrain_xrcc, terrain_yrcc, xpos, ypos, yaw, world_size, scale)
                                     
    # 6) Update worldmap (to be displayed on right side of screen)
    data.worldmap[obstacle_y_world, obstacle_x_world, 0] = 255
    data.worldmap[rock_y_world, rock_x_world, 1] = 255
    data.worldmap[navigable_y_world, navigable_x_world, 2] = 255
    
    # 7) Make a mosaic image, below is some example code
        # First create a blank image (can be whatever shape you like)
    output_image = np.zeros((img.shape[0] + data.worldmap.shape[0], img.shape[1]*2, 3))
        # Next you can populate regions of the image with various output
        # Here I'm putting the original image in the upper left hand corner
    output_image[0:img.shape[0], 0:img.shape[1]] = img

        # Let's create more images to add to the mosaic, first a warped image
    warped = perspect_transform(img, source, destination)  
        # Add the warped image in the upper right hand corner
    output_image[0:img.shape[0], img.shape[1]:] = warped
    
    # lower left, threhold image
    ret, thresh1 = cv2.threshold(warped, 160, 255, cv2.THRESH_BINARY)

    output_image[img.shape[0]:img.shape[0]*2, 0:img.shape[1]] = thresh1
        
        # Overlay worldmap with ground truth map
    map_add = cv2.addWeighted(data.worldmap, 1, data.ground_truth, 0.5, 0)
        # Flip map overlay so y-axis points upward and add to output_image 
        # world map, lower right
    output_image[img.shape[0]:, img.shape[1]:img.shape[1]+data.worldmap.shape[1]] = np.flipud(map_add)

        # Then putting some text over the image
    cv2.putText(output_image,"Rover Cam", (20, 20), 
                cv2.FONT_HERSHEY_COMPLEX, 0.4, (255, 255, 255), 1)
    cv2.putText(output_image,"Birds Eye View", (340, 20), 
                cv2.FONT_HERSHEY_COMPLEX, 0.4, (255, 255, 255), 1)
    cv2.putText(output_image,"Threshold", (20, 170), 
                cv2.FONT_HERSHEY_COMPLEX, 0.4, (255, 255, 255), 1)
    cv2.putText(output_image,"World Map", (340, 170), 
                cv2.FONT_HERSHEY_COMPLEX, 0.4, (255, 255, 255), 1)
    data.count - 1 # Keep track of the index in the Databucket()
    
    return output_image

Run `process_image()` on your test data using the `moviepy` functions provided to create video output of your result. 
And another! 
Done.  The video is located in the output folder and titled test_mapping.mp4  Below is a screenshot of that video.

![alt text][image4]

### Autonomous Navigation and Mapping

#### 1. Fill in the `perception_step()` (at the bottom of the `perception.py` script) and `decision_step()` (in `decision.py`) functions in the autonomous mapping scripts and an explanation is provided in the writeup of how and why these functions were modified as they were.
#### 2. Launching in autonomous mode your rover can navigate and map autonomously.  Explain your results and how you might improve them in your writeup.  

The perspective transform transforms the Rover image to a top down perspective.

An image with a grid from the Rover’s camera was taken while in the Rover simulator. 

The image was loaded into Jupyter notebook and plotted the image so the pixel’s coordinates could be retrieved

The perspective transform transforms the Rover image to a top down perspective.

An image with a grid from the Rover’s camera was taken while in the Rover simulator. 

The image was loaded into Jupyter notebook and plotted the image so the pixel’s coordinates could be retrieved. 
 
Using the grid, the four corners of a grid is identified to define what one square meter looks like on the ground.


![alt text][image6]


The function cv2.GetPerspectiveTransform was used to calculate the perspective transform matrix m.

The matrix M was applied to the warped function to generate a warped image.

Source, is the source points of the four corners on the grid of the original image that marked the four corners.  The destination points that are defined is the location of where the source points should be moving in the final image.  When the code is run, the image (birds eye) is presented with a top down view of the world

Notes:  The perspective transformation requires a 3x3 transformation matrix.  Straight lines will remain straight even after the transformation.  This process requires 4 points on the input image with corresponding points on the output image.    The perspective transformation is performed by the function cv2.getPerspectiveTransform then applying cv2.warpPerspective.
Perspective transform and color threshold can be switched. 
 
 ![alt text][image7]
 
 
Color threshold was applied to the picture to convert the image from a color image (3 tuple) to a single-channel binary image, where each pixel is set to one or zero.  Any pixel above the threshold is assigned a value of 1 while those below are assigned a value of 0. 
The color thresholded image is the navigable terrain in from of the rover. 
 
 
 ![alt text][image8]
 
 
Rover-Centric Coordinates – 

Pixel position of the white pixels were extracted and transformed where the rover camera is at (x,y) = (0,0).  

The coordinates were switched from y, x to x,y. 

Yaw angle is at zero when the rover is at x, y = 0,0

Yaw angle is measured counterclockwise.

Rotation is to account for the fact that the camera may take a picture that can be pointing in any direction via its yaw angle (rotate to make sure x,y are parallel to the axes.  Translation help to account for the rover may be located at any position when it takes a picture (translate rotated position by the x and y position).  The 

    yaw_rad = yaw * np.pi / 180

    x_rotated = xpix * np.cos(yaw_rad) - ypix * np.sin(yaw_rad)
    
    y_rotated = xpix * np.sin(yaw_rad) + ypix * np.cos(yaw_rad)

A rotation matrix is applied to the rover space pixel value.

A translation is performed by adding the x and y components.  

The components are divided by 10 to scale it down to give us world coordinates.  

The values are truncated to be within the range of the map.
 
The perspective transform transforms the Rover image to a top down perspective.

First we took an image with a grid from the Rover’s camera while in the Rover simulator. 

We loaded the image into Jupyter notebook and plotted the image so the pixel’s coordinates could be retrieved. 
 
Using the grid, the four corners of a grid is identified to define what one square meter looks like on the ground.
 
The function cv2.GetPerspectiveTransform was used to calculate the perspective transform matrix m.

The matrix M was applied to the warped function to generate a warped image.

Source, is the source points of the four corners on the grid of the original image that marked the four corners.  The destination points that are defined is the location of where the source points should be moving in the final image.  When the code is run, the image (birds eye) is presented with a top down view of the world

Notes:  The perspective transformation requires a 3x3 transformation matrix.  Straight lines will remain straight even after the transformation.  This process requires 4 points on the input image with corresponding points on the output image.    The perspective transformation is performed by the function cv2.getPerspectiveTransform then applying cv2.warpPerspective.
Perspective transform and color threshold can be switched. 
  
We convert x and y pixel positions to polar coordinates to determine the navigable terrain.  Each pixel is represented by the distance form the origin and angle counterclockwise.    Because the direction(angle) represents the average angle of all navigable terrain pixels in the rover’s field of view is roughly 0.7 radians in the plot above.  We convert degrees and clip to the range of +/-15 for steering.


![alt text][image9]


Improvements:
One improvement would be to address the challenge of mapping in uneven terrain.  In the real world, the ground rises and dips.  Currently, the mapping in this project only addresses the pixel in the ground plane. 
 

Another improvement would be to add a function that records the coordinates of the starting point and tracks where the rover has been.  This can be used to prevent the rover from retracing its steps and to ensure that in a larger environment, the rover can not only map the whole map but additional functionality can be added in to prevent the rover from looping.  


**Note: running the simulator with different choices of resolution and graphics quality may produce different results, particularly on different machines!  Make a note of your simulator settings (resolution and graphics quality set on launch) and frames per second (FPS output to terminal by `drive_rover.py`) in your writeup when you submit the project so your reviewer can reproduce your results.**

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

The functions and examples provided above is the approach that I took.  I am pursuing this project further using real cameras and sensors to apply it in real life.  I will try utilizing different sensors to find a solution to the problem of reading lane markers that are wet to simulate rainy conditions when it will be harder for the vehicle to see the markers and faded out lane markers to simulate construction zone areas where there are many markers on the ground that make it difficult for the vehicle to read the marker.
