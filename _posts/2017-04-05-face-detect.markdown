---
layout:     post
title:      "使用opencv进行人脸检测"
subtitle:   "使用python+opencv进行人脸检测、人眼检测等功能"
date:       2017-04-05 12:00:00
author:     "zfl"
header-img: "img/post-sklearn-svm.jpg"
header-mask: 0.6
catalog:    true
tags:
    - python
    - 机器学习
    - 人脸检测
    - opencv
    - python库
---

# 使用python库opencv实现人脸检测
## 安装opencv
1. 安装opencv的python库
```
    pip install opencv
```
2. 因为要使用opencv的haar分类器，所以去 [opencv官网](http://opencv.org/)下载Sources文件，opencv\sources\data\haarcascades路径下有已经训练好的分类器。关于haar分类器，这里有一篇非常好的博客[浅析人脸检测之Haar分类器方法](http://www.cnblogs.com/ello/archive/2012/04/28/2475419.html)深入浅出了描述了其基本原理，很值得拜读。

## 人脸检测
- 主要代码已经添加了注释：  

```python
import cv2
from PIL import Image,ImageDraw

def detectFaces(imagePath):
    imgData = cv2.imread(imagePath)
    face_cascade = cv2.CascadeClassifier(
        'C://my_program/opencv/sources/data/haarcascades/haarcascade_frontalface_default.xml')
    # 如果img维度为3，说明不是灰度图，先转化为灰度图gray；如果不等于3，原图就是灰度图
    if imgData.ndim == 3:
        gray = cv2.cvtColor(imgData, cv2.COLOR_BGR2GRAY)
    else:
        gray = imgData

    # 主要函数参数说明：
    # scaleFactor为每一个图像尺度中的尺度参数，默认值为1.1；
    # scale_factor参数可以决定两个不同大小的窗口扫描之间有多大的跳跃，这个参数设置的大，则意味着计算会变快，但如果窗口错过了某个大小的人脸，则可能丢失物体。
    # minNeighbors参数为每一个级联矩形应该保留的邻近个数，默认为3。
    #minNeighbors控制着误检测，默认值为3表明至少有3次重叠检测，我们才认为人脸确实存。
    faces = face_cascade.detectMultiScale(
        gray, scaleFactor=1.06, minNeighbors=3)

    # 将得到的识别结果在图像上画框标出
    img = Image.open(imagePath)
    draw_instance = ImageDraw.Draw(img)
    for (x, y, width, height) in faces:
        draw_instance.rectangle(
            (x, y, x + width, y + height), outline=(255, 0, 0))
    img.show()
```  

- 这也许是世界上最著名、传播最广的自拍了：  

    ![oscars.jpg](http://upload-images.jianshu.io/upload_images/730879-b2373a03e79f5944.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- 人脸检测的结果：  

    ![result.jpg](http://upload-images.jianshu.io/upload_images/730879-b80c91ed5f63e3b3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 通过不断调整detectMultiScale的两个参数可以得到不同的检测结果。其中scaleFactor必须大于1，当这个参数越小，计算量也就越大，通过实验检测到的人脸也就越多，同时其他物体被检测为人脸的几率也就越大；minNeighbors这个参数越小，重叠检测的次数也就越少，通过实验发现检测到的人脸越多，随之其他物体被检测为人脸的几率就越大。

## 人眼识别
- 主要代码以及注释：
```python
def detectEyes(imagePath):
    imgData = cv2.imread(imagePath)
    # 首先进行人脸检测，因为眼睛都在人脸上，然后再人脸上进行人眼检测，也可以直接进行人眼检测
    face_cascade = cv2.CascadeClassifier(
        'C://my_program/opencv/sources/data/haarcascades/haarcascade_frontalface_default.xml')
    if imgData.ndim == 3:
        gray = cv2.cvtColor(imgData, cv2.COLOR_BGR2GRAY)
    else:
        gray = imgData
    faces = face_cascade.detectMultiScale(
        gray, scaleFactor=1.06, minNeighbors=3)
    eye_cascade = cv2.CascadeClassifier(
        'C://my_program/opencv/sources/data/haarcascades/haarcascade_eye.xml')
    img = Image.open(imagePath)
    draw_instance = ImageDraw.Draw(img)
    # 对检测到的脸进行人眼检测，在进行画框的时候要注意相对位置
    # 参数和人脸检测相同
    for (x, y, width, height) in faces:
        roi_gray = gray[y:y + height, x:x + width]
        eyes = eye_cascade.detectMultiScale(
            roi_gray, scaleFactor=1.05, minNeighbors=2)
        for (ex, ey, ewidth, eheight) in eyes:
            draw_instance.rectangle(
                (x+ex, y+ey, x + ex + ewidth, y + ey + eheight), outline=(255, 0, 0))
    img.show()
```
- 人眼检测的结果：  

![result.jpg](http://upload-images.jianshu.io/upload_images/730879-84c9b9c51eb91a90.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 还有好多只眼睛没有检测出来哦！
## 总结
- 相对的还有许多其他的haar分类器，比如说笑脸检测、左眼检测、右眼检测、猫脸检测（可以用这个在加上树莓派做一个逗猫的机器人，只要检测到猫就去攻击它，哈哈）、身体检测、半身检测等，有时间的话也研究一下。