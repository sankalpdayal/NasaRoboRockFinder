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

[image1]: ./output/warped_thresh.png
[image2]: ./output/rock_identified.png
[image3]: ./output/direction.png 
[image4]: ./output/output_image.jpg 
[image5]: ./output/sub_global_map.jpg 
[image6]: ./output/output_image_max_dist_max_dir.jpg 
[image7]: ./output/output_image_min_dir.jpg 
[image8]: ./output/output_image_min_dist.jpg 
[image9]: ./output/final_state.jpg 


## [Rubric](https://review.udacity.com/#!/rubrics/916/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  

You're reading it!

### Notebook Analysis
#### 1. Run the functions provided in the notebook on test images (first with the test data provided, next on data you have recorded). Add/modify functions to allow for color selection of obstacles and rock samples.
I used the same logic as taught in lessons to perform thresholdoing to detect navigable terrain. I extended the existing function by adding an option if pixel values above the threshold is chosen or below. Above option is used for detection of navigable terrain and below for obstructions.
An example of warped and thresholded image is
 
![alt text][image1]

To detect rock pixels I created a new function `color_thresh_range()` which takes in a max threshold and a min threshold for colors and returns the pixels which lie within this range. The function is defined in the section Color Thresholding of notebook Rover_Project_Test_Notebook.ipynb. 
An example of binary image with pixels identified as rocks is

![alt text][image2]

The binary thresholded image of navigable terrain is used for estimating direction for rover. Same method as taught in the class is used. The complete pipeline from image to warped image to thresholded image and then to direction is shown here

![alt text][image3]


#### 2. Introduction of **sub global direction** to go towards unexplored areas

New method is added to help navigate the rover in **Not yet explored Areas**. This is done in two steps
1. First a sub global map is obtained using the global map and current position and direction of rover.
2. Using this sub global map, the pixel values defining the nature of terrain is used to determine a global direction. Highest weight is given to those areas where all 3 pixel values are 0. Which means the area is not explored yet.  

The function `get_sub_global_map()` gives a sub global map at the current position of rover and pointing in the direction faced by rover. It does following
1. Takes input the latest updated world map till now. (World map has pixel values R for obstruction, G for rocks and B for movable path. If all are 0, then that area is not explored yet.)
2. Takes input scale (size of sub global map) and creates a mask of x, y positions with x,y = 0,0 as center of rover and y<0 as left, y>0 as right and x>0 as front.
3. Then rotate and translate this mask of positions using current rover position and direction. This gives x,y positions in the world map.
4. Fetch the pixel values corresponding to x,y positions from the world map.
5. Return the pixel values as weights and x and y positions in the rover coordinates.

These pixel values will be used to get direction. Highest weight is given if the value is 0, which means the area is not explored yet. The function is written under the section Global Direction of notebook Rover_Project_Test_Notebook.ipynb.

Another function `get_sub_global_direction()` uses the pixel values of the sub global map as weight for the direction corresponding to that pixel. Sub global direction is calculated in following manner
1. Add the pixel values for obsturction and movable terrain.
2. Subtract this value from 255 and normalize it to 1. So if the sum of pixels is 0, then weight is 1.
3. If the weight is <0.95 then remove those pixels under considerations.
4. If there is no pixel left then give current direction as 0 else take weighted mean.
This gives a direction which is biased towards areas which are not yet explored enough.  An example of instance during the run of rover and the sub global map at that instance is shown here

Following image demonstrates an instance during the run. In the image
1. Top left shows the current scene as seen by the rover.
2. To right is the warped image
3. Bottom left is the world map as explored till now.
4. Bottom right is the direction chosen for the rover to navigate. In this the green arrow is the sub global direction, red arrow is the local direction and solid white arrow is the final chosen direction.
 
![alt text][image4]

Sub global map at that instance

![alt text][image5]

#### 3. Merging of **sub global direction** and **local direction**

The idea of merging global and local direction is using the concept that when the space in front of rover is high, then rover can follow the global direction, where as when the space in front of rover is constricted then rover follows the local direction. 
The judgement on how much current space is open or constricted can be done using the mean distance and standard deviation of direction of possible direction vectors in current scene as seen by rover. The function works in following manner. 

1. If the *mean_dir_sub_global* is 0, this means there is no global direction, hence make control parameter to 0.
2. Else, normalize distance between 0 to 1. If distance is high means there is enough space in front to navigate.
3. Normalize std between 0 to 1. If std deviation is high, this means there are many possible direction to choose from.
4. Hence to get final control parameter, mulitply these two. So that the final parameter is 1 only when space is also high and possible directions to choose is also high.

Also normalized rover speed is multiplied. This allows the control parameter high only when rover speed is high. This further make sures that rover has been moving freely. 

Final fusion is done as **final_direction = (mean_dir_sub_global)(control_param) + (mean_dir)(1-control_param)**. The function to perform this task is given under the section Merge Global and Local direction in the same notebook.

Following is that instance when both distance and direction are high. In this scenario, the rover has many options to choose among local direction, hence the weight for global direction tends to 1. 

![alt text][image6]

Following is that instance when the distance is high, but there arent many directions to choose from because path is too tight. Hence factor due to standard deviaton is 0 and hence the weight for global direction tends to 0.

![alt text][image7]
 
Following is that instance when the distance is low and there isnt much space to move. Hence factor due to distance is 0 and hence the weight for global direction tends to 0.

![alt text][image8]

#### 4. Populate the `process_image()` function with the appropriate analysis steps to map pixels identifying navigable terrain, obstacles and rock samples into a worldmap.  Run `process_image()` on your test data using the `moviepy` functions provided to create video output of your result. 
Process image performs the complete pipeline for processing of image. After creating warped binary thresholded image for navigable terrain, it estimates the local direction. Using the current positions and yaw of rover and the global map upated till now, estimates the sub global map and gets sub global direction.
Using the standard deviation and distances of navigable terrain in local map, gets the control parameter or weight to fuse the two directions. The intermediate values are shown on the top left of video.
1. First value is frame number
2. Second value is mean distance
3. Third value is standard deviation of angles multiplied by 100.
4. Fourth value is control parameter on a scale of 1 to 10.

In each frame in video 
1. Top left shows the current scene as seen by the rover.
2. To right is the warped image
3. Bottom left is the thresholded image for navigable terrain.
4. Bottom right is the direction chosen.

Video showing the pipeline to get direction on a pre recorded manual run of rover

[![Results on Manual Run of Rover](http://img.youtube.com/vi/qgExWvuIuWI/0.jpg)](http://www.youtube.com/watch?v=qgExWvuIuWI)


### Autonomous Navigation and Mapping

#### 1. Fill in the `perception_step()` (in `perception.py`) and `decision_step()` (in `decision.py`) functions in the autonomous mapping scripts.
In general, all the additions done in the notebook were ported to python scripts. The porting included functionality related to color thresholding, obtaining sub global map and estimating control parameter or weight. 
Other than these, there are two more additions done to the python scripts which are as follows
1. Getting the final direction for rover using the local direction, global direction and weight.
2. Introduction of a new mode for rover `stop_obs`. The rover goes into this mode if it is has been throttling for a few seconds but the velocity is still close to 0. This can happen if there a small rock in the path of rover. To get out of this state, 
rover rotates till the yaw has changed more than 30 degrees since it was in `stop_obs` mode. This way rover finds a way new way around the rock.

More specific updates in each script is given here.

Following updates were done in the script `perception.py`
1. Introduced option to choose above or below threshold in function `color_thresh()` to detect navigable terrain and obstruction using the the same function.
2. Added function `color_thresh_range()` to detect pixels containing rocks in an image.
3. Added function `get_sub_global_map()` to get the sub global map from current position and yaw or rover and latest updated global map.
4. Specifically in the function `perception_step()` following modifications are done.
	1. Completed pipeline to get warped binary thresholded image for navigable terrain, obstructions and rocks.
	2. Obtained the corresponding world map positions for these pixels and updated the `Rover.worldmap` accordingly.
	3. Using the binary image of navigable terrain, obtained the possible distances and directions.
	4. Using the current location and yaw of rover, obtained the current sub global map. 
	5. Added logic to use the pixle values as weights and get the sub global dierection.
	
Following updates were done in the function `decision_step()` (in `decision.py`)
1. Added logic to obtain mean local direction.
2. Added logic to get control parameter or weight for sub global direction using the mean of distances and standard deviation of angles.
3. Updated steering direction using the local direction, sub global direction and weight.
4. Added method to keep track of last 10 throttle values and velocity in an array.
5. Introduced condition to check if the mean throttle for last 10 values is high and mean velocity for last 10 values is still low and if check is true, then change mode to `stop_obs` and store the yaw. Also stop throttle and steer and set brakes.
6. Introduced condition to get out of `stop_obs` by checking if either velocity is high enough or if yaw has changed at least 30 since the rover went into the `stop_obs` mode. If the velocity is low and yaw has changed more than 30 deg, then change the mode back to 'forward'
  
Following updates were done in initialization of structure `Rover_state()` in script `drive_rover.py` to store more information
1. Introduced variable `dir_global` to store sub global direction estimated in `perception_step()` (in `perception.py`) and to be used in `decision_step()` (in `decision.py`) 
2. Introduced array `throttle_speed` to store last 10 throttle values and velocity. This are updated and used in `decision_step()` (in `decision.py`) 
3. Introduced variable `current_yaw` to store yaw of rover when it changes mode from `forward` to `stop_obs`. This is also updated and used in `decision_step()` (in `decision.py`) 

For debugging, I introduced display of intermediate variables Rover.mode, mean throttle and mean velocity in last 10 seconds in the command prompt. These changes were done in function `update_rover()` in script `supporting_functions.py`. 

#### 2. Launching in autonomous mode your rover can navigate and map autonomously. 

Since running the simulator had different choices of resolution and graphics quality and output can have different frames per second that can result in variations in results, I have noted these and are given in following table listing the details for the configuration which I had.

Setting|Configuration
------------- | -------------
|Operating System| Windows 7 64 bit
|Core| Intel i7 2.6 GHz 
|Screen| 640 x 480
|Graphics Quality| Good
|FPS| 12(Min), 25(Max), 15(Typical)

The last automonous run result had about **Mapped 98%** with **Fidelity 66.1%** and **Located 5** rocks. Following image is the screen shot of output when I terminated the run.

![alt text][image9]
 
The images for the last run are stored in the [folder](./auto_dataset3). The video stream of the complete run is here
 
[![Results of Autonomous Run of Rover](http://img.youtube.com/vi/CzpICfFCbXw/0.jpg)](http://www.youtube.com/watch?v=CzpICfFCbXw)


#### 3. Thought process, problems and possible scope of improvement
To challenge myself I didnt look at the walk through video as it was mentioned it had a solution and completed parts on my own. With just the basic navigation rover was not exploring new areas. 
This I felt because it didnt have a global contetxt. If a human had to perform this task he/she will first see which areas have been explored and which are not. Then whenever there is a turn or
technically more space to navigate, will choose a direction which will lead to un explored area. Hence to introduce this intelligence, I added a concept of have a sub global map that gives a direction which is biased towards the areas that are
not explored yet. To measure if there is enough space to manuever I introduced a control parameter that increased when there is more space. After tuning to what scale a subset of global map has to be considered and to what extent a space is considered 
enough navigable to consider more options for direction, the concept worked well to explore new areas.

Also I noticed that sometimes the rover was getting stuck in areas where there were rocks in pathway. This is an issue because the direction obtained was somewhat straight and still there were enough visible pixels. This was not letting the rover to go
in stop mode and rotate. To solve this I introduced a new mode - stopped due to obstruction. In obstruction I noticed that rover was giving its full throttle but speed wasnt increasing. Hence I introuced a mathematical way of monitoring this and put the rover into
stop due to obstruction mode. And to get out of this mode, simple solution was to change yaw a little and try moving again. This worked well to help rover to get unstuck.

Possible improvements that I would make is to make the global scale adaptive such that first it explores more near by areas, then it increases the scales and starts exploring farther areas and keep doing this till it has explored complete map. Also
luckily in this most of the offshoot segements are stright but if they had curvature or turns, it would be a good idea to define the map area as connected segements. And lastly I would like robot to pick rocks and then finally returns to the center.
 This can be done by first putting the rover in a new mode - going to pick up rock. During this mode, introduce logic for 
estimating a distance (in global coordinates) between rock center and rover position. The rover will speed up, slow down and stop in way to achieve an ideal distance. This will allow rover to pick up. 
Once the rock is picked the rover can again go back in forward mode. 





