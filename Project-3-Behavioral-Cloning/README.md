Udacity Self-Driving Car Project 3: Behavioral Cloning

<b>PROBLEM:</b> <br>
Use Behavioral Cloning to train a Convolutional Neural Network model with Keras, to drive a car in a simulator.


<b>SOLUTION:</b><br>
The problem was approached in these steps:<br>
1- gather data (images and steering angle values)<br>
2- analyze and modify the images and related steering angles<br>
3- create several Convolutional Neural Networks and choose the best one<br>
4- fine-tune the model (if needed)<br>

1- DATA:<br>
Initially I used to drive around the car in the simulator in “training mode” to collect data (images and values). I made several attempts but the keyboard’s response was most of the time very abrupt, creating not good data.
The approach was to drive smoothly for a couple of laps and then record “recovery actions”, teaching the car how to recover in case of going off the road.
Eventually Udacity provided a training dataset and I decided to use that one in order to mitigate any risk of feeding bad data. A joystick would have been a good solution as well.


2- IMAGE PREPROCESSING and LABELS AUGMENTATION:<br>
The images from the simulators have size (160x320) pixels, which are a bit big to be handled in big quantities.
Additionally some areas of the image were not useful and would have created more noise, “distracting” the network. 
That’s why I decided to crop the top of the image and a bit of the button portion. I tried several options, but I finally settled with cropping the height to [32:135] and keeping the full width [0:320].
Additionally I resized the image to (32x64) in RGB.

In regards to the labels I performed some augmentation since I have used left and right camera images as well, in addition to the center images.
The augmentation for the left and right images entailed to adjust the values by a factor of 0.15 (adding it in case of left images and subtracting it for the right images).
The reason for that is that the cameras of the two sides record an image that is off the center line, therefore we want to adjust the steering angle in order to “bring” it closer to the center.
So the image on the left would need to steer to the right a little bit (hence +0.15 to the current steering angle), while the image to the right would need to steer to the left (hence -0.15 to the current steering angle).
The value I used has been modified several times. I started with a more “gentle” approach (+-0.08) but I noticed that it produced no significant effects. Later I moved it to 0.1, then tried 0.2, but I finally used 0.15 as it looked to be a good compromised.

 To create additional data and to avoid the bias on the left turn (as most the track steers to the left) I added mirror images (with corresponding mirrored steering angle, multiplying the value by -1).
Finally the images and labels are split in training and validation sets. The test is performed running the simulator on the track in autonomous mode.
The test needed to be repeated several times as it’s described in the section 4.
Ultimately the images and steering angle value are fed to the network in batches, through the use of a generator (which allowed to load the data in batches instead of laoding all of them into memory).

3- CONVOLUTIONAL NEURAL NETWORK:<br>
I tried different model architectures. Initially I took inspiration from the NVIDIA model which had a network consisting of 9 layers, including a normalization layer, 5 convolutional layers
and 3 fully connected layers.
I had to adjust the kernel sizes and the strides in the convolutional layers since I was using smaller image sizes. (i.e. instead of kernel 5x5 with 2x2 strides I had 3x3 kernels with 2x2 strides in the first convolutional layer and then 3x3 kernel with 1x1 studies for the following ones). 
This network did not give me promising results so I modified it to the following:

    model = Sequential()
    model.add(Lambda(lambda x: x/127.5 - 1., input_shape=(32, 64, 3)))
    model.add(Convolution2D(24, 3, 3, border_mode='valid', subsample=(2,2), activation='relu'))
    model.add(Convolution2D(36, 3, 3, border_mode='valid', subsample=(1,1), activation='relu'))
    model.add(Convolution2D(48, 3, 3, border_mode='valid', activation='relu'))
    model.add(Convolution2D(64, 2, 2, border_mode='valid', activation='relu'))
    model.add(Convolution2D(64, 2, 2, border_mode='valid', activation='relu'))
    model.add(Flatten())
    model.add(Dense(512))
    model.add(Activation('relu'))
    model.add(Dropout(.2))
    model.add(Activation('relu'))
    model.add(Dense(10))
    model.add(Activation('relu'))
    model.add(Dense(1))

This network gave me good results but I went on and tried several other ones. I ended up using the following:

    model = Sequential()
    model.add(Lambda(lambda x: x / 127.5 - 1., input_shape=(32, 64, 3)))
    model.add(Convolution2D(32, 3, 3, border_mode='same'))
    model.add(LeakyReLU())
    model.add(Convolution2D(32, 3, 3))
    model.add(LeakyReLU())
    model.add(MaxPooling2D(pool_size=(2, 2)))
    model.add(Dropout(0.5))
    model.add(Convolution2D(64, 3, 3, border_mode='same'))
    model.add(LeakyReLU())
    model.add(Convolution2D(64, 3, 3))
    model.add(LeakyReLU())
    model.add(MaxPooling2D(pool_size=(2, 2)))
    model.add(Dropout(0.5))
    model.add(Flatten())
    model.add(Dense(512))
    model.add(LeakyReLU())
    model.add(Dropout(0.5))
    model.add(Dense(1))

I will explain the layers within this network.
The first layer is the lambda layer which regularizes the input images.
The following two layers are Convolution layers with 3x3 kernels (and standard stride = 1x1 (also called “subsample” in the Keras documentation).
Each one has an activation function, which in my case is a LeakyRELU. I tried using other ones, such as RELU and ELU, but eventually I kept the LeakyRELU, which is a special version of a Rectified Linear Unit that allows a small gradient when the unit is not active.
Then it follows a MaxPooling layer with a pool_size of 2x2.
This layer allows us to reduce the feature map size.(A convolution with strides of few pixels helps reduce the feature map size but it also removes information. On the other hand MaxPooling, at every point on the feature map, looks at small neigh borough around that point and computes the maximum of all the responses around it).
A Dropout layer is added after MaxPool, to avoid overfitting. A dropout forces to learn a redundant representation for everything. For this reason I used 0.5 instead of a lower value.
Further on I added 2 more Convolutional layers (and corresponding activation function after each one) with same filter sizes, but with higher number of filters (64 versus the 32 on the first two convolutional layers).
Another MaxPool layer is then added, followed by another Dropout.
Finally there is a Flatten layer in between the dropout and the Dense layer (with 512 as output dimension), in order to present a 2D array as an input to the Dense layer.
Then again an activation LeakyRELU function and a dropout layer, prior to the final single output.

The model parameters result as:

![Alt text](https://cloud.githubusercontent.com/assets/13647664/21406075/b94b5806-c797-11e6-9571-6aff694a7d2c.png)


I used Adam optimizer and I tried different learning rates, but eventually I kept the standard one (lr=0.002).
The network was run on several epochs, but I made sure to print the resulting model and weights for each epoch in order to test the performance.
I used the 7th epoch, which also resulted having the lowest value of mean squared error.

![Alt text](https://cloud.githubusercontent.com/assets/13647664/21406366/40f3de3a-c799-11e6-9ac6-9f7cb3eae549.png)


Although this model and weights were the best of a bunch, the car was not performing properly, meaning it was failing to approach the first right hand curve of the track (while instead going smoothly for the rest of it).
I tried running different epochs on the same initial dataset, but I ended up moving to the following step: fine-tuning.


4- FINE TUNING:<br>
Fine-tuning revealed to be very tricky. 
The approach was this one: use the model and weights saved during training and recompile and train the model with new data taken from the spot where the car failed to perform. 
So I ran the car in that curve and gathered these new images. I made several attempts as sometimes the new data was either not improving the performance or further degrading it.
I put model.py in fine-tune mode so to use the saved model and weights and ran it through 5 epochs.
I had hard time fine-tuning initially as I was using a very low learning rate. I tried to change it and eventually I was able to adjust the behavior of the car with a learning rate of 0.001.
The fine tuning was hard as many factors could influence it: new training data, number of epochs, learning rate.


CONCLUSIONS:<br>
Training data are key to a good performance but are not all. The network architecture is also relevant, although sometimes it’s hard to arrive to a “final” model. Creating a network is an art and a science. I used several networks for different projects and I certainly realized that a network performing well on a dataset is not certain to do the same on a different one.
It’s very important to preprocess data. I worked on a project for image recognition of the SVHN dataset and even there I experienced the importance of feeding “clean” data to the network.
In this case, cropping the images was definitely impactful, as it removed a lot of noise.
Also augmentation in this case was really helpful.
I worked on this project mostly using my CPU, but it revealed to be very inefficient. There are a lot of things that need to be adjusted and you want to try several options. Having to wait minutes for training the network is very frustrating.
After few days I got my AWS set up and used the GPU which was very fast and very efficient in making progress and in removing frustration.
Finally, fine-tuning is definitely really tricky. In my case it worked best when I gathered few images on the specific area of the track where the car was going off road. Also I tried to put the car in the same position where it was heading and I started recording the new images from there, adjusting immediately the steering angle towards the center lane.

A VIDEO of the Autonomous driving mode can be seen here:<br>
[![Autonomous driving](http://img.youtube.com/vi/cdMVPxF6kw4/0.jpg)](http://www.youtube.com/watch?v=cdMVPxF6kw4)

______________

<i>Files used:</i><br>

<b>model.py</b> - The script used to create and train the model. <br>
<b>drive.py</b> - The script to drive the car.<br>
<b>model.json</b> - The model architecture.<br>
<b>model.h5</b> - The model weights.<br>


