---
title: "A Computer Vision Problem"
date: 2022-03-05
description: "CV Task given as an assignemnt in ARK"
tags: ["tech"]
---

{{< lead >}}
An interesting Computer Vision problem with all my approaches and an elegant solution.
{{< /lead >}}

Original problem statement file and required files [here](https://drive.google.com/drive/folders/1BDYyxglyihcEPfUyxoK2Iq1567Ymt3PN?usp=sharing). This problem was given to me in my sophomore year as a task under my tenure in the ARK perception team.

I haven't done robotics in a long while (am writing this thing after leaving all that stuff behind), this being the only testimony of me having done those courses on image processing lol.

## Problem Statement

>If none of the following make sense to you and those text files seem like random jargon, this post is **NOT** for you. Go and move on, you're better off not knowing about all this altogether (unless you want to pursue a career in robotics, maybe)

A breif version of the problem goes like this-

- given 30 files, 10 each of types 0 (RGB, jpg), 1 (Segmentation, jpg) and 2 (Pose, txt)
- images are 256x144 in size
- a projective pinhole camera is used everywhere
- camera has a horizontal field-of-view (FOV) of 90°
- assumptions-
  - the distortion parameters are zero
  - principal point of the image is at its center
  - focal length in x direction = focal length in y direction
- pose data is the position of the camera in the world coordinates in the NED coordinate system
- it is stored in the form of an orientation quaternion and a position vector

### Objective

Using the information provided, calculate

1. length of the side of the cube
2. x,y,z coordinates of the centroid of the cube

{{< alert >}}
**WARNING**: Proceed ahead only after you're ready for the solution (after giving the problem some thought) and are already familiar with the basics of image processing and Computer Vision
{{< /alert >}}

## Initial Attempt

I started to recall everything that I'd gone through while doing my courses. Thought about applying all sorts of stuff like homography, constructing the camera matrices, using stereo pairs (seemed illogical at that time).

My line of thought only seemed to go in one direction- use all images separately.

But then I hit a dead wall when I couldn't convert the 2D coordiantes to 3D ones (was the essence of the entire problem). I thought maybe the information given was insufficient and I needed a ```scale``` factor.

The whole 3D -> 2D conversion eats up the depth information completely, hence a (x, y, z) -> (x', y') mapping exists. However to do the opposite, you need to materialize the depth information from somewhere.

But then again, I wasn't using all the information provided to me, hinting towards my shortcoming.

## My solution approach

I started to think of some use case to the segmentation masks provided. I had to locate corners in tha image according to image coordiantes. So there it was, I used the segmentation masks, ran them through the Harris corner detector function of OpenCV.

The results were good honestly,

but

## Result

---
