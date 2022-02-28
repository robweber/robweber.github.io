---
layout: post
title:  "EPD Image Processing"
author: robweber
categories: coding
tags: epd python omni-epd
---

This is mostly a reminder for myself but perhaps could be useful for others in the same boat. In short, image processing is a pain. This is especially true when dealing with devices, like e-ink displays, that use limited color sets. Black and white conversions are fairly easy to deal with but as soon as you start adding multiple colors things can get tricky.

Below are some of my notes on dealing with Tri-color Waveshare devices and how to process the images. For reference I'll be using code from my [omni-epd][omni-epd] project, which implements these image processing methods. The library used throughout is the excellent [Pillow image processing library][pillow]. If you want to play with any of these methods I've written a [filter script][filter-gist] that can both filter an image based on a given palette, or save the colors separate as shown below. 

<!--more-->

{% include toc %}

## Background

One of my personal projects is [omni-epd][omni-epd], an abstraction library for E-ink displays (EPD) so that multiple types of displays, from different manufacturers, can all be communicated with through the same set of Python class objects. This is super useful as even devices from the same manufacturer have differences in method calls and parameters from display to display. With `omni-epd` you can simply pass in the name of the device and interact with it through a common set of functions such as:

```python

epd.prepare()
epd.display()
epd.clear()
epd.close()

```

## Adjusting the Image Palette

A common problem when dealing with these displays is taking your full color image and turning it into something that the EPD can display with a limited color palette. The easiest of this is just turning something into a black and white image.

```python
from PIL import Image

image = Image.open('/path/to/file.png')

# convert to BW
image = image.convert(mode="1")

# save to see results
image.save('/path/to/file_bw.png', "PNG")

```

This code simply opens and image and uses the built in `convert()` method to turn it into a BW image. The argument `mode` can convert an image into a number of [different image types](https://pillow.readthedocs.io/en/stable/handbook/concepts.html#concept-modes); but `1` is just a b/w image. Full details on `convert()` are in the [Pillow docs](https://pillow.readthedocs.io/en/stable/reference/Image.html#PIL.Image.Image.convert).

Where things get trickier is if you have an EPD that can handle more than one color. Some devices, like the [Inky Impression](https://github.com/pimoroni/inky), can simply take the raw `Image` object and do the conversion for you. For Tri-color Waveshare devices though you need to filter out the colors you don't want from the image first. This is done by adjusting the image palette.

```python
from PIL import Image

total_colors = 3  # we are filtering out 3 colors from the main image

"""palette as an array of RGB values
white - 255,255,255
black - 0,0,0
red - 255,0,0"""
palette = [255, 255, 255, 0, 0, 0, 255, 0, 0]

# create a new image to define the palette
palette_image = Image.new("P", (1, 1))

# apply the palette to the image
# image has total 256 colors, set the rest to black
palette_image.putpalette(palette + [0, 0, 0] * (256-total_colors))

# load the source image
image = Image.open('/path/to/file.png')

if(image.mode != 'RGB'):
  # convert to RGB as quantize requires it
  image = image.convert(mode='RGB')

# apply the palette
image = image.quantize(palette=palette_image)

# save to see results
image.save('/path/to/file_filtered.png', "PNG")

```

One important thing to note is the setting of all the other colors to black. This will ensure that you get the rest of the detail from this image, but rendered as black. Using the above a very simple example function can easily be written to take an array of RGB tuples `[(255,255,255), (0,0,0)]`, generate the correct palette filter image, and apply it to a base image. The result of the filter above can be seen in the image below.

| ![original image](/images/2022-02-25/test.jpg)  | ![Black and Red filtered image](/images/2022-02-25/test_filtered.png) |
| Original image | Black and Red filtered |

## Sending Color Images to WaveShare

So now we have an image that is stripped down to the colors the display will support. The final trick is sending the image to the display. This isn't as straightforward as it would seem since the Tri-color Waveshare displays take 2 `Image` objects. The first represents what is drawn for the black color and the second represents what should be drawn for the second (red or yellow) color. This means that yet again we need to separate out the images into essentially two b/w images.

```python
from PIL import Image

first_color = (0, 0, 0, 255, 255, 255)
second_color = (255, 255, 255, 255, 255, 255, 0, 0, 0)

# assume we have the 3 color image from the example above
img_black = image.copy()
img_color = image.copy()

img_black.putpalette(first_color)
img_color.putpalette(second_color)

# send to Waveshare
epd.display(epd.getbuffer(img_black), epd.getbuffer(img_color))

```

Taking a look a what each of those `Image` objects looks like you'll see they are both b/w images; however the first one represents the black color from the three color image and second has the red coloring from the three color image. These are then sent as parameters to the EPD driver which draws them as black and red on the screen. The `first_color` and `second_color` tuples are just setting every other color to white except for the one we want to remain.

| ![Black filtered image](/images/2022-02-25/test_black.png) |  ![Red filtered image](/images/2022-02-25/test_red.png) |
| Black Filtered | Red Filtered |

If you're really astute you'll see that the b/w image is actually color inverted. It's drawing the black as white and the white as black. This is because the `display()` function will take the image buffer and invert it back. It's a weird quirk of how Waveshare does the display but important nonetheless.

## Conclusion

The guts behind this confuse me every time I have to look at it. I've never been great at this sort of programming but having this as a reference in the future will help me refresh my memory.

## Links

* [filter_color.py][filter-gist] - script to split images based on palette using code in this post
* [omni-epd][omni-epd]
* [Pillow][pillow]
* [Waveshare Driver Source](https://github.com/waveshare/e-Paper)

[omni-epd]: https://github.com/robweber/omni-epd
[pillow]: https://pillow.readthedocs.io/en/stable/index.html
[filter-gist]: https://gist.github.com/robweber/08c79e0f0eb5f239f4b50209291702a3
