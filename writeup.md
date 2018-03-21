## Project: Search and Sample Return
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
[image_grad]: ./calibration_images/example_grid1.jpg
[image_rock1]: ./calibration_images/example_rock1.jpg
[image_rock2]: ./calibration_images/example_rock2.jpg
[image_rock3]: ./calibration_images/example_rock3.jpg
[image_rock4]: ./calibration_images/example_rock4.jpg
[image_find]: ./misc/Find_rock.jpg

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

Most of the modified codes basically follow "Project Walkthrough" instruction. Ref[1]

### Notebook Analysis
#### 1. Run the functions provided in the notebook on test images (first with the test data provided, next on data you have recorded). Add/modify functions to allow for color selection of obstacles and rock samples.
a) Find rock  
In the calibration_images, there are two kinds of rock, one is in the dark side and on in the bright side.
Base on this two picture, I cut it and remained the rock rock part. By using the find_rock_thresh function to figure out 
how to set the threshold. Base on the value, I still need to tweak it a little bit. Ref[2]  
![alt text][image_rock3]  
![alt text][image_rock4]

```sh
def find_rock_thresh(img):
    for c in range(0, 3): 
        tmp_im = np.zeros(img.shape) 
        tmp_im[:,:,c] = img[:,:,c] 
        one_channel = img[:,:,c].flatten() 
        print("channel", c, " max = ", max(one_channel), "min = ", min(one_channel))

# Create rock3&4 base on rock1&2 and get the color domain
example_rock3 = '../calibration_images/example_rock3.jpg'
example_rock4 = '../calibration_images/example_rock4.jpg'
    
rock3_img = mpimg.imread(example_rock3)
rock4_img = mpimg.imread(example_rock4)

print("Rock in light side")
find_rock_thresh(rock3_img)

print("Rock in dark side")
find_rock_thresh(rock4_img)
```
```sh
Rock in light side
channel 0  max =  215 min =  125
channel 1  max =  196 min =  106
channel 2  max =  118 min =  0
Rock in dark side
channel 0  max =  254 min =  106
channel 1  max =  215 min =  86
channel 2  max =  84 min =  0
```
```sh
def find_rock(img, upper_thr=(255, 215, 70),lower_thr=(105, 85, 0)):
    #set the channel 2 to 70 because 80 is too high 
    color_select = np.zeros_like(img[:,:,0])

    thresh = (img[:,:,0] > lower_thr[0]) \
           & (img[:,:,1] > lower_thr[1]) \
           & (img[:,:,2] > lower_thr[2]) \
           & (img[:,:,0] < upper_thr[0]) \
           & (img[:,:,1] < upper_thr[1]) \
           & (img[:,:,2] < upper_thr[2]) \
                
    color_select[thresh] = 1     
    
    return color_select
```
![alt text][image_find]

b) obstacle  
In perspect_transform, I created a mask for the non observed area and used it to get the obstacle map

```sh  
     mask = cv2.warpPerspective(np.ones_like(img[:,:,0]), M, (img.shape[1], img.shape[0]))# keep same size as input image
```
```sh 
    obs_map = np.absolute(np.float32(threshed)-1) * mask
```
#### 1. Populate the `process_image()` function with the appropriate analysis steps to map pixels identifying navigable terrain, obstacles and rock samples into a worldmap.  Run `process_image()` on your test data using the `moviepy` functions provided to create video output of your result. 


```sh
def process_image(img):
        # Define source and destination points & Apply perspective transform
    warped, mask = perspect_transform(img, source, destination)
        # Apply color threshold 
    threshed = color_thresh(warped)
        # Convert obstacles
    obs_map = np.absolute(np.float32(threshed)-1) * mask
        # Convert rover-centric pixel values to world coords
    world_size = data.worldmap.shape[0]
       
        # Get position from data
    scale = 2 * dst_size
    xpos = data.xpos[data.count]
    ypos = data.ypos[data.count]
    yaw = data.yaw[data.count]
    
        # Convert thresholded values to rover-centric coords
    ter_x_pix, ter_y_pix = rover_coords(threshed)
    ter_x_world, ter_y_world = pix_to_world(ter_x_pix, ter_y_pix, xpos, ypos, yaw, world_size, scale)
        # Convert obstacles values to rover-centric coords
    obs_x_pix, obs_y_pix = rover_coords(obs_map)
    obs_x_world, obs_y_world = pix_to_world(obs_x_pix, obs_y_pix, xpos, ypos, yaw, world_size, scale)
      
    data.worldmap[ter_y_world, ter_x_world, 2] = 255
    data.worldmap[obs_y_world, obs_x_world, 0] = 255
    nav_pix = data.worldmap[:, :, 2] > 0
    data.worldmap[nav_pix, 0] = 0

        # Find rock
    rock_map = find_rock(warped, upper_thr=(255, 215, 70),lower_thr=(105, 85, 0))
    if rock_map.any():
        rock_x, rock_y = rover_coords(rock_map)
        rock_x_world, rock_y_world = pix_to_world(rock_x, rock_y, xpos, ypos, yaw, world_size, scale)
        data.worldmap[rock_y_world, rock_x_world, 1] = 255

        # First create a blank image (can be whatever shape you like)
    output_image = np.zeros((img.shape[0] + data.worldmap.shape[0], img.shape[1]*2, 3))
        # Next you can populate regions of the image with various output
        # Here I'm putting the original image in the upper left hand corner
    output_image[0:img.shape[0], 0:img.shape[1]] = img

        # Let's create more images to add to the mosaic, first a warped image
    warped, mask = perspect_transform(img, source, destination)
        # Add the warped image in the upper right hand corner
    output_image[0:img.shape[0], img.shape[1]:] = warped

        # Overlay worldmap with ground truth map
    map_add = cv2.addWeighted(data.worldmap, 1, data.ground_truth, 0.5, 0)
        # Flip map overlay so y-axis points upward and add to output_image 
    output_image[img.shape[0]:, 0:data.worldmap.shape[1]] = np.flipud(map_add)

        # Then putting some text over the image
    cv2.putText(output_image,"Populate this image with your analyses to make a video!", (20, 20), 
                cv2.FONT_HERSHEY_COMPLEX, 0.4, (255, 255, 255), 1)
    if data.count < len(data.images) - 1:
        data.count += 1 # Keep track of the index in the Databucket()
    
    return output_image
```
### Autonomous Navigation and Mapping

#### 1. Fill in the `perception_step()` (at the bottom of the `perception.py` script) and `decision_step()` (in `decision.py`) functions in the autonomous mapping scripts and an explanation is provided in the writeup of how and why these functions were modified as they were.

Base on the notebook, I changed the following iteams for the perception_step, but didn't change the decision_step.
1. Img to Rover and other related image name
2. Removed the output_image since it is not necessary
3. Addjust the value for Rover.worldmap
```sh
# Apply the above functions in succession and update the Rover state accordingly
def perception_step(Rover):
    # Perform perception steps to update Rover()
    # TODO: 
    # NOTE: camera image is coming to you in Rover.img
    # 1) Define source and destination points for perspective transform
    # 2) Apply perspective transform
    # 3) Apply color threshold to identify navigable terrain/obstacles/rock samples
    # 4) Update Rover.vision_image (this will be displayed on left side of screen)
        # Example: Rover.vision_image[:,:,0] = obstacle color-thresholded binary image
        #          Rover.vision_image[:,:,1] = rock_sample color-thresholded binary image
        #          Rover.vision_image[:,:,2] = navigable terrain color-thresholded binary image

    # 5) Convert map image pixel values to rover-centric coords
    # 6) Convert rover-centric pixel values to world coordinates
    # 7) Update Rover worldmap (to be displayed on right side of screen)
        # Example: Rover.worldmap[obstacle_y_world, obstacle_x_world, 0] += 1
        #          Rover.worldmap[rock_y_world, rock_x_world, 1] += 1
        #          Rover.worldmap[navigable_y_world, navigable_x_world, 2] += 1

    # 8) Convert rover-centric pixel positions to polar coordinates
    # Update Rover pixel distances and angles
        # Rover.nav_dists = rover_centric_pixel_distances
        # Rover.nav_angles = rover_centric_angles
    
    # The destination box will be 2*dst_size on each side
    dst_size = 5 
    # Set a bottom offset to account for the fact that the bottom of the image 
    # is not the position of the rover but a bit in front of it
    # this is just a rough guess, feel free to change it!
    bottom_offset = 6
    image = Rover.img
    source = np.float32([[14, 140], [301 ,140],[200, 96], [118, 96]])
    destination = np.float32([[image.shape[1]/2 - dst_size, image.shape[0] - bottom_offset],
                    [image.shape[1]/2 + dst_size, image.shape[0] - bottom_offset],
                    [image.shape[1]/2 + dst_size, image.shape[0] - 2*dst_size - bottom_offset], 
                    [image.shape[1]/2 - dst_size, image.shape[0] - 2*dst_size - bottom_offset],
                    ])

        # Define source and destination points & Apply perspective transform
    warped, mask = perspect_transform(image, source, destination)
        # Apply color threshold 
    threshed = color_thresh(warped)
        # Convert obstacles
    obs_map = np.absolute(np.float32(threshed)-1) * mask
        # Convert rover-centric pixel values to world coords
    world_size = Rover.worldmap.shape[0]
       
    Rover.vision_image[:,:,2] = threshed * 255
    Rover.vision_image[:,:,0] = obs_map * 255

        # Get position from data
    scale = 2 * dst_size
 
        # Convert thresholded values to rover-centric coords
    ter_x_pix, ter_y_pix = rover_coords(threshed)
    ter_x_world, ter_y_world = pix_to_world(ter_x_pix, ter_y_pix, Rover.pos[0], Rover.pos[1], Rover.yaw, world_size, scale)
        # Convert obstacles values to rover-centric coords
    obs_x_pix, obs_y_pix = rover_coords(obs_map)
    obs_x_world, obs_y_world = pix_to_world(obs_x_pix, obs_y_pix, Rover.pos[0], Rover.pos[1], Rover.yaw, world_size, scale)
      
    Rover.worldmap[ter_y_world, ter_x_world, 2] +=30
    Rover.worldmap[obs_y_world, obs_x_world, 0] +=1

    dist, angles = to_polar_coords(ter_x_pix,ter_y_pix)
    Rover.nav_angles = angles

        # Find rock
    rock_map = find_rock(warped, upper_thr=(255, 215, 70),lower_thr=(105, 85, 0))
    if rock_map.any():
        rock_x, rock_y = rover_coords(rock_map)
        rock_x_world, rock_y_world = pix_to_world(rock_x, rock_y, Rover.pos[0], Rover.pos[1], Rover.yaw, world_size, scale)
        Rover.worldmap[rock_y_world, rock_x_world, 1] = 255

        rock_dist, rock_angles = to_polar_coords(rock_x,rock_y)
        rock_idx = np.argmin(rock_dist)
        rockx = rock_x_world[rock_idx]
        rocky = rock_y_world[rock_idx]

        Rover.worldmap[rocky,rockx,1] = 255
        Rover.vision_image[:,:,1] = rock_map * 255
    else:
        Rover.vision_image[:,:,1] = 0
    return Rover
```
#### 2. Launching in autonomous mode your rover can navigate and map autonomously.  Explain your results and how you might improve them in your writeup.  
Results can pass the requirement.
The following items could improve in the feture
1. When the rover hit the rock or wall, it can't get away.
2. Finish the pick rock.
3. Modify the "decision" function to make more state for decision making.
4. When in it goes circle, it should identify the condition and escape.

Test condition is 1920*1080 with good quality with 33 FPS

Reference:
Ref[1]  Project Walkthrough 
Ref[2] https://ddnews.me/tech/zmc5e4e6.html

**Note: running the simulator with different choices of resolution and graphics quality may produce different results, particularly on different machines!  Make a note of your simulator settings (resolution and graphics quality set on launch) and frames per second (FPS output to terminal by `drive_rover.py`) in your writeup when you submit the project so your reviewer can reproduce your results.**
