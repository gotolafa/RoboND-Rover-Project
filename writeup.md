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

[image1]: ./misc/rover_image.jpg
[image_grad]: ./calibration_images/example_grid1.jpg
[image_rock1]: ./calibration_images/example_rock1.jpg 
[image_rock2]: ./calibration_images/example_rock2.jpg 
[image_rock3]: ./calibration_images/example_rock3.jpg 
[image_rock4]: ./calibration_images/example_rock4.jpg 
[image_find]: ./misc/Find_rock.jpg
---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

You're reading it!

### Notebook Analysis
#### 1. Run the functions provided in the notebook on test images (first with the test data provided, next on data you have recorded). Add/modify functions to allow for color selection of obstacles and rock samples.
a) Find rock
    In the calibration_images, there are two kinds of rock, one is in the dark side and on in the bright side.
Base on this two picture, I cut it and remained the rock rock part. By using the find_rock_thresh function to figure out 
how to set the threshold. Base on the value, I still need to tweak it a little bit. 
![alt text][image_rock3][image_rock4]

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
And another! 

![alt text][image2]
### Autonomous Navigation and Mapping

#### 1. Fill in the `perception_step()` (at the bottom of the `perception.py` script) and `decision_step()` (in `decision.py`) functions in the autonomous mapping scripts and an explanation is provided in the writeup of how and why these functions were modified as they were.


#### 2. Launching in autonomous mode your rover can navigate and map autonomously.  Explain your results and how you might improve them in your writeup.  

**Note: running the simulator with different choices of resolution and graphics quality may produce different results, particularly on different machines!  Make a note of your simulator settings (resolution and graphics quality set on launch) and frames per second (FPS output to terminal by `drive_rover.py`) in your writeup when you submit the project so your reviewer can reproduce your results.**

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  



![alt text][image3]


