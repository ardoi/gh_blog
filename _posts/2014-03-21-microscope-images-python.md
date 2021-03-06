---
title: Python & microscope images
layout: post
nav: blog
comments: true
---

A few years ago I worked in a lab that had a Zeiss confocal microscope. My job was to conduct experiments and analyze images obtained in these experiments. The latter I like to do in Python, but to my surprise, no way to read Zeiss images into Python existed. Luckily for me, my supervisor had managed to obtain the specifcation for the Zeiss binary format. With the help of this sacred document I wrote a [reader for Zeiss files][1] and got on to analysing my data.

Year later I'm in a different lab. Here the people have built a microscope of their own, but thankfully use TIFF format to save their files. At least there are [libraries][2] to read TIFF files in Python. Equipment and acquisition parameters were written as custom formatted text in the description field of the TIFF file. Parsing that was an order of magnitude easier than implementing a binary file reader. So a bit of tinkering but analysis was soon up and running. 

Cue forward a few years and I'm now a postdoc in yet another lab. This place has an Olympus confocal microscope but no specification file for the binary blob that images get saved to. Even having the spec file  would only solve a part of the problem. We had the idea of releasing some analysis software to the scientific community and it wouldn't really be very helpful if it could only read one or two types of microscope images. Implementing binary readers for all would have required format spec files, sample data files and time, none of which I had.

## Java saves they day
Turned out that reinventing the wheel was not necessary. The *de facto* ruler of image analysis tools for biosciences, [ImageJ][3], already had a way of reading in a [myriad of different commercial file formats][5]. It accomplishes this by using the [Bio-Formats plugin][4]. So, instead of trying to accommodate all possible file formats myselaf, it was way easier to use Bio-Formats which can convert any microscope image into [OME format][6] and then read that with Python.

##PyMImage 
###(short for Python Microscope IMage)
Loading an image is as easy as this:
{% highlight py %}
>>> import pymimage.imagemaker as imm

#create an ImageMaker using .ome_folder for 
#storing generated OME files (folder does not
#have to exist)
>>> imaker = imm.ImageMaker(".ome_folder")
#load an Olympys image file
>>> image_file = imaker.load_file("~/data/awesome_experiment.oib")
#take a look at the image properties in the file
>>> print image_file.image_attr
{0: {'channels': 1,
  'data_type': 'uint16',
  'frames': 1,
  'image_height': 10000,
  'image_step_y': 0.276,
  'image_step_x': 0.276,
  'image_width': 344},
 1: {'channels': 1,
  'data_type': 'uint16',
  'frames': 1,
  'image_height': 512,
  'image_step_y': 0.276,
  'image_step_x': 0.276,
  'image_width': 512}}
#read in the first image
>>> image_file.read_image(0)
>>> im0_data = image_file.images[0]['ImageData']
>>> print im0_data.shape, im0_data.mean()
((1, 1, 10000, 344), 1989.4566779069767)
{% endhighlight %}

And we have the data from our microscope image file! As you can see, the image data array has 4 dimensions. These are for channel, frame, x and y dimensions. For a single 2D framescan channel and frame count will be 1, as above. 

In case you have multiple images in a folder you can either load the entire folder with `imm.load_dir(...)` or a selection of files with `imm.load_files(...)`. File conversion for multiple file is executed in parallel so doing a whole bunch of files together will be a lot faster than running them one by one. 


##Getting it
Clone from [github][7] and run the setup
{% highlight bash %}
git clone https://github.com/ardoi/pymimage.git
pip install -e .
{% endhighlight %}
A zip archive of the package is also [available][8]. You'll need to have `numpy` installed. 

## How does it work
This is not really important if you just want to load images but might not hurt to know.

###Conversion
Each image is converted to OME with the Bio-Formats `bfconvert` tool. This means `java` needs to be installed on your machine (and it probably is). The conversion to OME takes a few seconds (or more, depending on the size of your image), but only has to be done **once**. Next time you load the same image the existing OME file will be read from the cache directory you specify when creating an ImageMaker  without having to perform the conversion again. The generated OME file is pure XML and can be parsed with standard Python libraries. 

###Format specific information
In some cases you might want to read extra information from the file. For example, in linescan mode, LSM images store the time of acquisition for each line in an image. In OME files these are stored as annotations.
{% highlight py %}
{'ObjectiveLens NAValue': '1.2',
 'ObjectiveLens Name': 'UPLSAPO  60X W  NA:1.20',
 'PMTDetectingMode': 'Analog',
 'PMTVoltage': '700',
 'PinholeDiameter': '800000',
 'PinholeScale': '1',
 'Resolution': '10.0',
 'RotationValue': '0.0',
 'SamplingClock': '250000',
 'ScanSpeed': '4.0',
 'Time Per Frame': '24784272.000',
 'Time Per Line': '2472.000',
 'Time Per Pixel': '4.0',
 'Time Per Series': '24784272.000',
 'X Pinhole': '-143',
 'XPanValue': '-236',
 'Y Pinhole': '-447',
 'YPanValue': '209',
 'ZoomValue': '150',
 ...etc...
 }
{% endhighlight %}

For accessing this extra information consistently for each file format the `CustomReader` class can be subclassed. In the function `_get_typespecific_extra_info` anything extra can be read from the annotations and made available. Right now readers for Zeiss, Olympus and VTI files exist. 

You might say that how is this different from the original problem of having to write separate readers for each file format? The big difference is that with the OME approach most, if not all, microscope file formats can be imported even without a specific `CustomReader`. The generic `OMEXMLReader` can get most of the important data out by itself. Only for some nonstandard stuff are these extra readers necessary. Also, it's orders of magnitude easier to figure out what to extract from an OME-XML file than trying to squeeze it out from a binary blob.






[1]:https://code.google.com/p/lsjuicer/source/browse/inout/reader.py?name=0.2rc2
[2]:https://code.google.com/p/pylibtiff/
[3]:http://rsb.info.nih.gov/ij/
[4]:http://downloads.openmicroscopy.org/bio-formats/5.0.0/
[5]:http://www.openmicroscopy.org/site/support/bio-formats5/supported-formats.html
[6]:http://www.openmicroscopy.org/Schemas/Documentation/Generated/OME-2013-06/ome.html
[7]:https://github.com/ardoi/pymimage
[8]:https://github.com/ardoi/pymimage/releases/tag/v0.1
