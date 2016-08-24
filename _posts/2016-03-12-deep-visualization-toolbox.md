---
layout: post
title: Caffe & Deep Visualization Toolbox
---

I attended a meetup and talk by Gene Kogan and was completely fascinated by the visualization tools used by him for few of his computer vision examples. I looked for lot of tools that help to visualize Neural Networks. Finally I bumped into Deep Visualization ToolBox.

A lot of times features in neural networks in computer vision (specifically Convolution Neural Networks) are hard to imagine. Deep neural networks have been mostly thought of as “black boxes” and their inner workings are mysterious and inscrutable. While going through most of the famous CNNs that have increased accuracy in image classification, you would come across description about the layers stating that the neurons in the Jth layer learn to detect edges, the pooling layer probably
relaxes the strictness of features in convolution inputs, however with little justification about how and why about the feature.

Deep Visualization ToolBox helps to visualize neuron by neuron features in layers where you can actually see that a particular layer is learning to detect the edges or another is learning to detect the light intensity.

Deep Visualization ToolBox uses already trained models in Caffe and creates synthetic image representations in each layer of the model. More fascinating details here.

I made several attempts to install Caffe and Deep Visualization ToolBox on my Ubuntu 15.10 system, one of them was disastrous enough to leave my system in a unusable state (Not good at linux, probably executed something really vicious from some forum comment).


### Caffe installation ###

Caffe installation page describes the details step by step and in details.However there were still few hiccups in between, solutions for which I had to google. I would recommend following the Caffe page for installation but in case you face any of the issues listed below which I faced, the notes might help you as well.

*  Follow the prerequisite for Ubuntu Systems here. All dependencies can be installed using the commands mentioned on the page. You might need to install CMake if you don’t have it installed already.
sudo apt-get install cmake

*  Clone Caffe from github. You need to work on a specific branch of Caffe for Deep Visualization Toolbox. Details are described on Github page.

*  Compile Caffe using the steps described here. Edit MakeFile.config to install Caffe in CPU mode only if you don’t have GPU.

*  If make in step 3 fails with “fatal error: hdf5.h: No such file or directory caffe”, you need to create a sym-link for hd5 libraries in /usr/lib/x86_64-linux-gnu.

{% highlight bash %}
$ cd /usr/lib/x86_64-linux-gnu
$ sudo ln -s libhdf5_serial.so.8.0.2 libhdf5.so
$ sudo ln -s libhdf5_serial_hl.so.8.0.2 libhdf5_hl.so
{% endhighlight %}

Also, add /usr/include/hdf5/serial in INCLUDE_DIR in Makefile.config

{% highlight bash %}
INCLUDE_DIRS := $(PYTHON_INCLUDE) /usr/local/include /usr/include/hdf5/serial
{% endhighlight %}

*  If make install / make -j8 fails with “fatal error: caffe/proto/caffe.pb.h: No such file or directory”. Running the following commands in Caffe directory should solve the issues.

{% highlight bash %}
$ protoc src/caffe/proto/caffe.proto –cpp_out=.
$ mkdir include/caffe/proto
$ mv src/caffe/proto/caffe.pb.h include/caffe/proto
{% endhighlight %}

*  Add /usr/lib/x86_64-linux-gnu/hdf5/serial/ to LIBRARY_DIR in Makefile.config is you face error : “/usr/bin/ld: cannot find -lhdf5_hl /usr/bin/ld: cannot find -lhdf5”

{% highlight bash %}
LIBRARY_DIRS:=$(PYTHON_LIB) /usr/local/lib /usr/lib /usr/lib/x86_64-linux-gnu/hdf5/serial
{% endhighlight %}

*  Make should finish successfully at this point.

### Deep Visualization ToolBox installation ###

Deep Visualization Toolbox is pretty simple as described in the github page.

*  Need to install the prerequisite opencv, scipy.

{% highlight bash %}
$ sudo apt-get install python-opencv
$ sudo apt-get install scipy
$ sudo apt-get install python-numpy python-scipy python-matplotlib \
    ipython ipython-notebook python-pandas python-sympy python-nose
{% endhighlight %}

*  Configure the toolbox using the steps described on the github page.

*  Fetch the models using fetch.sh. If the fetch fails with “Can’t create dir”, this is probably because of extra space at line 38 second argument. Fixing that should complete the download.

*  Disable GPU at line 30 caffevis_mode_gpu = False. 

Time to run the toolbox and visualize CNN layers!

### Sample Images ###

![Deep Visualization Ambulance](/img/deep_vis_amb.png)

![Deep Visualization Bottle](/img/deep_vis_bottle.png)
