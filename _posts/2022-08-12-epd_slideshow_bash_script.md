---
layout: post
title:  "EPD Slideshow Bash Script"
author: robweber
categories: coding
tags: epd bash omni-epd
---

Like a lot of people lately, I wanted to display some of the new images from the [James Webb Telescope][james-web-homepage]. I have an electronic paper display (EPD) connected to a [Raspberry Pi][raspberry-pi-homepage] on my desk and figured that would be a cool way to see them as a rotating slideshow. Not wanting to spend a lot of time configuring a special image viewer I hacked together a __26 line Bash script__ to rotate through the images. You can easily get this done in a variety of ways but this serves as a good example of using Open Source libraries to do some heavy lifting and making quick work of a simple idea.

![James Webb - Southern Ring Nebula](/images/2022-08/southern_ring_nebula.png)

<!--more-->

{% include toc %}

## Setup

This is a super simple Bash script, however there is some setup involved. To talk to the EPD I'm using the [omni-epd Python library][omni-epd], which I also happen to be the maintainer of. `omni-epd` is a Python library that abstracts the process of sending images to a variety of electronic paper displays. Critically for this quick project, it includes a test utility `omni-epd-test` that is installed as a command line testing tool when you install the library.

Assuming the EPD is connected to a Raspberry Pi the library and test utility can be installed with:

```bash
sudo pip3 install git+https://github.com/robweber/omni-epd.git#egg=omni-epd
```

Once installed you can run the test utility manually using the `omni-epd-test` command. For a list of supported devices, see the [omni-epd project page](https://github.com/robweber/omni-epd#displays-implemented).

```bash
usage: omni-epd-test [-h] -e EPD [-i IMAGE]

EPD Test Utility

optional arguments:
  -h, --help            show this help message and exit
  -e EPD, --epd EPD     The type of EPD driver to test
  -i IMAGE, --image IMAGE
                        Path to an image file to draw on the display
```

## Image Slideshow

Since `omni-epd` is doing the heavy lifting of putting the image on the display; all I really needed to do at this point was send it the path to an image.  Here is the full script but I'll break it down below.

```bash
#!/bin/bash -e
# epd_random_image.sh - display a random image, from a folder of images, on an EPD
# run the script with ./epd_random_image.sh /path/to/images
EPD="waveshare_epd.epd7in5_V2"

# setup some variables
DIR_PATH=$1
IMAGES=()

# get all PNG images in the path
for image in $DIR_PATH/*.png
do
  IMAGES=(${IMAGES[@]} "$image")
done

# get a random index
TOTAL=${#IMAGES[@]}
INDEX=$(($RANDOM % $TOTAL))

# UNCOMMENT LINES TO DEBUG
#echo "Path: $DIR_PATH"
#echo "Total images: ${TOTAL}"
#echo "Random Image: ${IMAGES[INDEX]}"

# display the image
omni-epd-test -e $EPD -i ${IMAGES[INDEX]}
```

### Breaking It Down

The first part of the script simply defines some variables, including the EPD type. The directory path is passed in as a command line variable when the script is run. To get a list of all images in the path I'm using the following loop below. It cycles through each file in the directory, filtering on PNG images. These are then added to the `IMAGES` array. Note that you have to take the existing array, add the new image, and then set the variable with each loop. If you don't do this you'll end up with just one image path at the end.

```bash
for image in $DIR_PATH/*.png
do
  IMAGES=(${IMAGES[@]} "$image")
done
```

Once the images are in the array we want to choose one randomly. For this I'm using the `RANDOM` bash function. You can try this function on the command line using `echo $RANDOM`. By default this will give you a random integer from 0 to 32767. You can define an upper bound though to limit the results. In this case it's just the max size of the `IMAGE` array.

```bash
TOTAL=${#IMAGES[@]}
INDEX=$(($RANDOM % $TOTAL))
```

This code will generate a value between 0 and the max number of images. Now that a random image in the array is found, the last line uses `omni-epd-test` to update the EPD.

## Conclusion

This probably took me 20 min total to put together the whole thing. I did have to do some "Googling" on how to do the array randomization but that was about it. Obviously this whole thing could be done with Python, or using a dedicated digital photo frame type of project. The Bash script was easy and can be setup on a timer with `cron`. When I'm sick of James Webb I can quickly turn it off and put the display back into service as a [very slow movie player](https://www.google.com/search?q=very+slow+movie+player).

## Links

* [James Web Telescope][james-web-homepage] - image gallery for James Webb Space Telescope
* [Raspberry Pi][raspberry-pi-homepage] - small computer board running Raspberry Pi OS (Linux)
* [Omni-EPD][omni-epd] - python library to control the writing of an image to an electronic paper display

[james-web-homepage]: https://webbtelescope.org/news/first-images/gallery
[raspberry-pi-homepage]: https://www.raspberrypi.com/
[omni-epd]: https://github.com/robweber/omni-epd
