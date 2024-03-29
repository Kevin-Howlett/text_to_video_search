# :eye: Hindsight AI

## Important!
This is a web application that depends on both a GUI and GPU acceleration. 
It is designed for use on a windows machine with a CUDA capable GPU.

## Usage  
Download the repo and create an anaconda env from the provided .yml file.
```bash
$ conda env create -n hindsight-gpu -f hindsight-gpu.yml
$ conda activate hindsight-gpu
$ pip install git+https://github.com/openai/CLIP.git
```
With the requirements installed and your hindsight-gpu env activated navigate to the repo directory.  
Place any videos you would like to analyze into the repo directory. To launch the app run:
```bash
$ streamlit run app.py 
```

## System Architecture
HindsightAI is a web application that allows users to query their own video content for objects, actions, and situations that occur within it. In order to accomplish this task our application leverages the power of OpenAI’s cutting-edge CLIP image classification model.

To use our application users begin by uploading an mp4 video file. In the background the provided video file is converted into a format expected by CLIP. Once the video file has been converted, users are able to type a query ranging from a single word to entire sentences and the application will then take the user to the point in the video that the algorithm determines is closest related to the query. Additionally, users are able to select from all of the results the algorithm finds, filter these results to only allow predictions of a certain confidence, as well as adjust the granularity of the algorithm to allow it to catch local ‘smaller’ details (feature in development).

The application is composed of two main components, the graphic user interface (GUI) and the video processing pipeline. The GUI (see Section 4) is powered by Streamlit - a python based web application development framework. The GUI allows users to upload, query, and interact with their video content. The video processing pipeline (see Section 2) is powered by decord and handles the pre-processing of videos as well as their subsequent analysis.

<img src="images/system_design.png?raw=true"/>   

Additionally, we plan to expand on the current functionality of the application by implementing a method in which folders of video files can be searched for user's queries. The app will perform a cursory search of each video and alert the user to the videos that are most likely to contain the desired subject. The user will then be able to select one of the videos and view the observed instances in a similar fashion to the single video search feature outlined above.

## Input/Output Processing
Video preprocessing is handled solely by our Video class. The initial method of the Video class ‘videoToTensor‘ takes both a relative path to an .mp4 video as well as a user-selected frame-rate value as inputs. The class then converts the provided video into individual frames and these individual frames are converted into tensors for later clip processing. The Video class is designed such that .mp4 videos of any size or frame rate will be usable for clip analysis. The frame rate argument provided to the ‘videoToTensor‘ method determines how often frames are sampled from the video with 1 frame per second being the default and the average framerate of the video (sampling every frame) being the maximum value. The resulting data structure is a dictionary of tensors where the key is the frame count and the value is the tensor of that frame.

The ‘clipAnalyze’ method of the Video class Runs a clip analysis on each frame of the provided dictionary's values where the possible text classes are the user provided string. Additionally, the method renders a streamlit progress bar that scales to the length of the analysis. This method takes a dictionary of tensors (output by ‘videoToTensor’) and a user-provided query in the form of a string as input. The method outputs a dictionary of probabilities where the key is the frame count and the value is the probability that the query provided by the user is related to that frame. The results are returned in descending order with the highest probability results appearing first in the dictionary. This method ties into our user-selectable prediction confidence widget that allows users to select the minimum probability result they want the model to return.

For the proposed batch video search function, a two step process will occur. The user will first upload a collection of video files and a desired search term.  The video files will be given a cursory search, on the fly, in which the highest confidence level for the term throughout the video is recorded in a dictionary structure. Once this dictionary structure is populated for every video in the batch, the user will be prompted with a list that contains the name of the video and a confidence level.  The user will then be able to select a video and analyze CLIP's observation in a similar manner as the single video process.  A new class will be developed for the batch processing that occurs in step one while the current methods will be utilized for step two.

## Introducing CLIP
The model we are utilizing in our application, CLIP (developed by OpenAI), is a generalized image classification model which can take any image and produce word embeddings for the purpose of matching raw text strings to the contents of the image. The design and training of the model allows for high zero-shot performance in classifying images (i.e. image classification problems outside of the training set). The following image provides a summary of the model (taken from A. Radford et al.):

<img src="images/clip.png?raw=true"/> 

While typical image classification models train an image feature extractor and a linear classifier to predict a label, CLIP trains an image encoder and text encoder to predict the correct pairings of a batch of (image, text) training examples. At test time the learned text encoder synthesizes a zero-shot linear classifier by embedding the names or descriptions of the target dataset’s classes.

The approach we are taking in utilizing CLIP in our application is running equally spaced frames of a video through the model, producing word embeddings for the entire video. We then take user input in the form of a text string query, calculate probabilities of matches between text and word embeddings for each of the frames, and subsequently deliver frames/timestamps of interest based on the user query. As part of our pipeline, we are also implementing frame segmentation to allow the user to adjust the level of clarity of each query. This takes the form of a custom function which takes in image tensors for frames of the video along with an argument for the level of clarity, and outputs smaller overlapping sub-images/tensors. The sub-images will then be fed to CLIP, word embeddings will be compared to text strings, and the highest probability match will be outputted.  

## The App

When running the app, as shown in the picture below, you will be prompted to submit a MP4 video. The user may then select a confdience threshold (defaulted to 85%), a frame rate at which they would like the analysis (lower frame rate means linearly faster analysis, at cost of missing information) and which analysis type/granularity level they would like. 

The granularity dropdown contains four options: "Low", "Medium", "High", and "Recursive". It is recommended to use recursive, as this will rcursively hone in on the exact spot, in each frame, which contains the best match. In the following example, the user has searched a music video for "man doing backflip". The app has intelligently found the exact spot in which this action occurs.

![](https://github.com/Kevin-Howlett/text_to_video_search/blob/main/images/Hindsight%20AI%20App%201.png)

![](https://github.com/Kevin-Howlett/text_to_video_search/blob/main/images/Hindsight%20AI%20App%202.png)

![](https://github.com/Kevin-Howlett/text_to_video_search/blob/main/images/Hindsight%20AI%20App%203.png)

The application also displays an interactive bar chart for all of the frames analyzed. This provides a more general scope of where matches were found in the video.

![](https://github.com/Kevin-Howlett/text_to_video_search/blob/main/images/Hindsight%20AI%20App%204.png)

We can see how the recursive analysis feature works through the following GIF. Each frame being analyzed is split by a (at first large) moving kernel and the sub-images are subsequently analyzed by the model. The kernel then beomes smaller by a predertmined proportion and further analysis is completed on the highest matching sub-image. This is done recursively until either no improvements are made in match confidence or the kernel becomes too small.

![](https://github.com/Kevin-Howlett/text_to_video_search/blob/main/images/hindsight_recursive.gif)

![](https://github.com/Kevin-Howlett/text_to_video_search/blob/main/images/hindsight_recursive.jpg)

The model is also good at matching more abstract concepts and content. As shown below, the user searched an episode of the cartoon TV show "SpongeBob Squarepants" for the phrase "muscles" and the application found a muscular cartoon fish.

![](https://github.com/Kevin-Howlett/text_to_video_search/blob/main/images/hindsight_muscle_fish.png)

