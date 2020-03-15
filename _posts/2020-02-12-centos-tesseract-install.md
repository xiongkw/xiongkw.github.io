---
layout: post
title: Centos-7安装Tesseract
categories: [编程, linux]
tags: [tesseract, leptonica]
---


> 

#### 1. 安装依赖包

```
$ yum install libstdc++ autoconf automake libtool autoconf-archive pkg-config gcc gcc-c++ make libjpeg-devel libpng-devel libtiff-devel zlib-devel
```

#### 2. 安装leptonica

> Leptonica is a pedagogically-oriented open source site containing software that is broadly useful for image processing and image analysis applications

```
$ tar zxvf leptonica-1.79.0.tar.gz
$ cd leptonica-1.79.0
$ ./autogen.sh
$ ./configure
$ make && make install
```

#### 3. 安装Tesseract

> `Tesseract`，一款由HP实验室开发由Google维护的开源OCR（Optical Character Recognition , 光学字符识别）引擎，特点是开源，免费，支持多语言，多平台

```
$ ./autogen.sh
$ export LD_LIBRARY_PATH=$LD_LIBRARY_PAYT:/usr/local/lib
$ export LIBLEPT_HEADERSDIR=/usr/local/include
$ export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig
$ ./configure --with-extra-includes=/usr/local/include --with-extra-libraries=/usr/local/include
$ make && make install
$ ldconfig
$ tesseract -v
tesseract 4.1.1
 leptonica-1.79.0
  zlib 1.2.7
 Found AVX2
 Found AVX
 Found FMA
 Found SSE
```

#### 4. 测试

```
$ tesseract test.jpg stdout -l eng
Estimating resolution as 124
123456
```

#### 5. 参考

* [Leptonica](http://www.leptonica.org/index.html)
* [Leptonica-release](https://github.com/DanBloomberg/leptonica/releases)
* [tesseract-release](https://github.com/tesseract-ocr/tesseract/releases)