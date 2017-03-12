# Project 05 - Vehicle Detection and Tracking
[![Udacity - Self-Driving Car NanoDegree](https://s3.amazonaws.com/udacity-sdc/github/shield-carnd.svg)](http://www.udacity.com/drive)

## README

[//]: # (Image References)
[image1]: ./output_images/car_not_car.png
[image2]: ./output_images/hogoutput.png
[image3]: ./output_images/sliding_windows.png
[image4]: ./output_images/sliding_window.jpg
[image5]: ./output_images/bboxes_and_heat.png
[image6]: ./output_images/labels_map.png
[image7]: ./output_images/output_bboxes.png
[image8]: ./output_images/out0.png
[image9]: ./output_images/out1.png
[image10]: ./output_images/out2.png
[image11]: ./output_images/out3.png
[video1]: ./project_video.mp4


### Histogram of Oriented Gradients (HOG)

#### 1. Explain how (and identify where in your code) you extracted HOG features from the training images.

The code for  loading the image is done using `glob` and it is coded in the IPython notebook file named `Traning.ipynb` in Cell number 3 and 4.
Here ,I have started by reading in all the `vehicle` and `non-vehicle` images.  Here is an example of one of each of the `vehicle` and `non-vehicle` classes:

![alt text][image1]

I then explored different color spaces and different `skimage.hog()` parameters (`orientations`, `pixels_per_cell`, and `cells_per_block`).  

 Here is an example using the Y channel of the `YUV` color space and HOG parameters of `orientations=8`, `pixels_per_cell=(8, 8)` and`cells_per_block=(2, 2)`:


![alt text][image2]

#### 2. Explain how you settled on your final choice of HOG parameters.

I tried various combinations of parameters and final configuration was chosen according to the one which gave the best test-set accuracy from the classifier (described later). Here is an example using the Y channel of the `YUV` color space and HOG parameters of `orientations=8`, `pixels_per_cell=(8, 8)` and `cells_per_block=(2, 2)`:

![alt text][image8]

*Note: OpenCV's `HOGDescriptor` class was used in place of `skimage.hog` as it was found to be significantly faster. The method `init_hog` in `training.py` initializes the descriptor with the parameters and then it is re-used for the rest of the process.*

#### 3. Describe how (and identify where in your code) you trained a classifier using your selected HOG features (and color features if you used them).

A `LinearSVC` was used as the classifier for this project. The training process can be seen in code cells 4-6 of `Project05-Training.ipynb`. The features are extracted and concatenated using functions in `training.py`. Since the training data consists of PNG files, and because of how `mpimg.imread` loads PNG files, the image data is scaled up to 0-255 before being passed into the feature extractor.

The features include HOG features, spatial features and color histograms. The classifier is set up as a pipeline that includes a scaler as shown below:

```python
clf = Pipeline([('scaling', StandardScaler()),
                ('classification', LinearSVC(loss='hinge')),
               ])
```
This keeps the scaling factors embedded in the model object when saved to a file. One round of hard negative mining was also used to enhance the results. This was done by making copies of images that were identified as false positives and adding them to the training data set. This improved the performance of the classifier. The model stored in the file `models/clf_9764.pkl` obtained a test accuracy of 98.85% where the test-set was 20% of the total data.
### Sliding Window Search

#### 1. Describe how (and identify where in your code) you implemented a sliding window search.  How did you decide what scales to search and how much to overlap windows?

The function `create_windows` in lines 141-147 of `detection.py` was used to generate the list of windows to search. The input to the function specifies both the window size, along with the 'y' range of the image that the window is to be applied to. Different scales were explored and the following set was eventually selected based on classification performance:

| Window size | Y-range     |
|-------------|-------------|
| (64, 64)    | [400, 500]  |
| (96, 96)    | [400, 500]  |
| (128, 128)  | [450, 600] |

![alt text][image3]
#### 2. Show some examples of test images to demonstrate how your pipeline is working.  What did you do to optimize the performance of your classifier?

The final image used the Y-channel of a YUV-image along with the sliding window approach shown above to detect windows of interest. These windows were used to create a heatmap which was then thresholded to remove false positives. This strategy is of course more effective in a video stream where there the heatmap is stacked over multiple frames. `scipy.ndimage.measurements.label` was then used to define clusters on this heatmap to be labeled as possible vehicles. This is shown in lines 48-56 of the  `process_image` function in `project05.py`. The heatmap is created using the `update_heatmap` function in `detection.py` in lines 171-186.

In the video pipeline (described in the next section), the output of the `label` function was used as measurements for a Kalman Filter which estimated the corners of the bounding box.

![alt text][image8]

![alt text][image10]

![alt text][image11]

![alt text][image9]
---

### Video Implementation

Video can  be found at:  [project_Video_out.mp4](project_Video_out.mp4)

Challage Video is available at  [test_video_out.mp4](./Output/test_video_out.mp4)

#### 2. Describe how (and identify where in your code) you implemented some kind of filter for false positives and some method for combining overlapping bounding boxes.


### Heatmap and Clustering 
I recorded the positions of positive detections as given by the Classifier in each frame of the video. To remove many spurious detection , I have thresholded decision function as given below 
```python
dec = clf.decision_function(test_features)
prediction = int(dec > 0.75)
```

## HeatMap 
Heatmap was added over 25 frame ,and any pixels below a thresholded of 10 were made zero . This help to exclude any false positive detection .

---
### Vehicle Tracking 

The process of tracking the vehicle, given a set of bounding boxes from the thresholded heatmap is performed by the `VehicleTracker` class in `tracking.py`. After the clusters were extracted from the heatmap using `scipy.ndimage.measurements.label`, the bounding boxes were passed in as measurements to a Kalman Filter.

The tracking process is mainly done by the `track` method of the `VehicleTracker` class. The first step in the process is to perform non-maximum suppression of the bounding boxes to remove duplicates.

The following rules were used to identify possible candidates for tracking. These are implemented in lines 182-208 of `tracking.py`.

1. If there are no candidates being tracked, assume all the non-max suppressed bounding boxes as possible tracking candidates.

2. If there are candidates currently being tracked, compute the overlap between the measured bounding boxes as well as the distance between the centroids of these boxes and the candidates.

3. The measured bounding boxes are assigned to a candidates based on the overlap as well as the distance between the centroids.

Once each measurement has been assigned to an existing (or new) candidate, the Kalman Filter is used to update the state of each candidate. The filter uses the x and y positions of the top-left and bottom-right corners of the bounding box as the state along with their second and third derivatives.


 From the positive detections I created a heatmap and then thresholded that map to identify vehicle positions.  I then used `scipy.ndimage.measurements.label()` to identify individual blobs in the heatmap.  I then assumed each blob corresponded to a vehicle.  I constructed bounding boxes to cover the area of each blob detected.  

Here's an example result showing the heatmap from a series of frames of video, the result of `scipy.ndimage.measurements.label()` and the bounding boxes then overlaid on the last frame of video:

#### Cleanup

The `cleanup` method in the `VehicleTracker` class removes stray vehicle candidates that haven't haven't been seen in a few frames or those that have gone beyond a pre-defined area of interest. This is implemented in lines 238-241 of `tracking.py`.

Each candidate also has an "age" value that is initially set to zero. Once the age hits `-50`, the variable is locked, and the  candidate is assumed to be valid vehicle detection and permanently tracked even if it is not seen for a few frames. The Kalman Filter estimates the velocity of the bounding box and predicts the position whenever the measurements are lacking. A permanently tracked vehicle is only deleted once it goes outside a certain region-of-interest (horizon). This helps maintain the tracking even when vehicles are obscured.

The is decremented by one any time the candidate has at least one measurement assigned to it. Conversely, the age is increased if no measurements are assigned to a candidate in a frame (assuming it's age > -50). Once the age reaches `5`, the candidate is considered a false positive and removed.

There is also a threshold on the age at which a candidate is drawn on the image. An orange bounding box indicates that the covariance estimated by the Kalman filter has increased beyond a certain limit. This happens when a vehicle candidate is obscured behind another and hasn't been detected for a few frames.

#### Discussion

Using of the Convolution neural network for detection of car and not car would be more accurate which might reduce false positive senario which will imporove the speed of computation .
  

