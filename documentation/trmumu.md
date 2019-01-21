---
layout: page
title: "Documentation: Transcoding on Raspberry Pi"
category : [documentation,transcoding_RPi]
description: ""
---
{% include JB/setup %}

Author: Guenter Kreidl

# October 2018 update

There is a new transcoder package for the Raspbery Pi, rtranscode 4.0, which does work on both Raspbian Jessie and Stretch.

Please have a look at my online manual for details: [Transcoding Manual](http://steinerdatenbank.de/software/rtranscode4_manual.pdf)

An overview and download instruction can be found in the [RPi forum](https://www.raspberrypi.org/forums/viewtopic.php?t=123876#p835946)

The latest RPi package package also contains MuMuDVB and a tutorial and some scripts to create a MuMuDVB backend and a simple frontend.



# Transcoding with the Raspberry Pi

The main use of transcoding is to provide a low bandwidth stream which can be served to a large number of clients or across a low bandwidth connection (your internet connection, for example). This is done by decoding the original dvb video stream, optionally reducing the image size and encoding it again, using a more efficient encoder (compared to MPEG 2, for example) or using a lower quality setting. More bandwitdh reduction can be achieved by reducing the number of audio channels and by also transcoding audio to a lower bandwidth.

The Raspberry Pi family of cheap ARM computer boards, priced from 5 (RPi Zero) to 35 USD (RPi 3), contains a GPU which is well suited for real time transcoding of DVB streams. It just needs the right kind of software.

A Raspberry Pi 3 (or the older model 2) is powerful enough to run mumudvb, oscam and the transcoding software at the same time, serving (and descrambling) a whole transponder. But transcoding is limited to one channel at a time.

If you are running mumudvb on another device, it may still be useful to add a Raspberry Pi to your network just for transcoding as it runs on low power (12.5 VA maximum).

Note: in all following examples I run mumudvb on the same Raspberry Pi (localhost), using unicast and port 9082.

My transcoding solution for the Raspberry Pi uses gstreamer-1.0 modules including the gstreamer-omx module which can access the GPU and use it for decoding and encoding of video streams. To serve the transcoded stream on a http server it uses a slightly extended version of Sebastian Droege's http-launch. The following example takes a mumudvb unicast stream SD stream (MPEG2 encoded video) and serves it on http://IP:9080/xyz.mkv as H264 encoded MKV live stream with an image size of 360x288, a video bitrate of 281 KBit and also transcodes one of the original audiostreams to 64 KBit AAC: 

`http-launch 9080 /xyz.mkv video/x-matroska default souphttpsrc location="http://localhost:9082/bysid/28106" is-live=true keep-alive=true do-timestamp=true retries=10 typefind=true blocksize=16384 ! tsdemux parse-private-sections=false name=demux demux.audio_0066 ! queue ! mpegaudioparse ! mpg123audiodec ! audioconvert dithering=0 ! audio/x-raw,channels=2 ! avenc_aac compliance=-2 bitrate=65536 ! matroskamux name=stream streamable=true demux. ! queue ! mpegvideoparse ! omxmpeg2videodec ! omxh264enc target-bitrate=288000 control-rate=variable ! video/x-h264,stream-format=byte-stream,profile=high,width=360,height=288,framerate=25/1 ! h264parse ! queue ! stream.`

Configuring and issuing such command lines would be a real pain and so I have written another application, rtranscode, which manages the use of http-launch in a much simpler way. In it's most basic useage it takes four command line arguments:

`rtranscode [options] uri videomode audiomode audiopid`

To get the same stream as above I run:  
`rtranscode http://localhost:9082/bysid/28106 sd1 mpeg 0x66`

You still need to know a bit about the original stream to run it. To find the arguments, rtranscode contains a stream analyzer. Running  
`rtranscode -g=http://localhost:9082/bysid/28106`  
will give you the channel name and the arguments:  
`Das Erste=http://localhost:9082/bysid/28106 sd1 mpeg 0x66`

Running  
`rtranscode -t=http://localhost:9082/bysid/28106`  
will not only find the arguments for you but will immediately start the transcoder stream.  

There are options to set the image size, video bitrate, audio bitrate and more. For example, this command  
`rtranscode -a=1 -v=0 -s=6 -t=http://localhost:9082/bysid/28106`  
will run the same stream with an image of size 720x576, using a video bitrate of 844 KBit and an audio bitrate of 32KBit (mono).

Here's another example, transcoding a 1080i HD stream:  
```rtranscode -a=1 -v=0 -h=3 -t=http://localhost:9082/bysid/108  
Starting to transcode  
Size: 910x512  VBR: 948K  ABR: 32K AC3  
Listening on http://127.0.0.1:9080/xyz.mkv```  

For more comfortable use you can run rtranscode in a simple (curses) menu mode. This requires a channel database file, which is a simple text file containing lines like this  
Das Erste=http://localhost:9082/bysid/28106 sd1 mpeg 0x66  
(which can be found using the -g option)

rtranscode contains another tool to create such channel databases. Here is a more complete example.

I'm serving the whole transponder 12187.50 from Astra S19.2E with mumudvb. First I download the playlist:  
`wget -O rtl.m3u http://localhost:9082/playlist.m3u`  
Now I use rtranscode to create a channel database:  
`rtranscode -i=rtl.m3u -o=rtl.dat`  
This will take a short while, because each stream is analyzed.  
Now I can run rtranscode with this channel database:  
`rtranscode -d=rtl.dat`  
The main menu will look like this:  

```ABR: 64K (a)  VBR: medium (v)  SD-Size: 360x288 (s)  HD-Size: 768x432 (h)  
Channels:  
RTL Television (0)  RTL Regional NRW (1)  RTL HB NDS (2)  
RTL FS (3)  RTL2 (4)  TOGGO plus (5)  
SUPER RTL (6)  VOX (7)  RTLNITRO (8)  
RTLplus (9)  n-tv (10)  RTL HH SH (11)```  
  
`Enter a channel number, 'a','v','s','h' or 'q' to quit:`  

You can start transcoding any channel by entering its number. You can use the a, v, s and h commands to select the audio bitrate, video bitrate and image size for SD or HD video.

That's not all you can do, but for everything else you should consult the README file or the manual (PDF).

Limits:

The transcoder is limited to what the GPU of the Raspberry Pi can do, which uses the same hardware for both decoding and encoding.

SD video (576i, both MPEG2 and MP4/H264 encoded streams) can be encoded to a maximum image size of 720x576 (original size).  
HD 720p50 video can be encoded to a maximum output size of 1024x576.  
HD 1080i video can be encoded to a maximum output size of 1280x720.  

Requirements:

You need a Raspberry Pi 2 or 3 (recommended) with a recent full Raspbian OS installed (usually on SD card).

For MPEG2 transcoding you will have to buy the MPEG2 license from the Raspbery Pi Foundation (2.5 $, MPEG decoder is disabled by default to reduce the SOC costs).

To make sure that all gstreamer modules are installed, run the following command:

`sudo apt-get install gstreamer1.0-libav gstreamer1.0-omx gstreamer1.0-plugins-bad
gstreamer1.0-plugins-base gstreamer1.0-plugins-base-apps gstreamer1.0-plugins-
good gstreamer1.0-plugins-ugly gstreamer1.0-tools`

For best performance we will overclock the GPU and set the GPU memory to at least 128 MB. Add the following to /boot/config.txt:

```gpu_freq=500  
force_turbo=1  
gpu_mem=128```  

(requires a reboot to work).

Installation:

```wget http://steinerdatenbank.de/software/transcoder3.tar.gz  
tar -xzf transcoder3.tar.gz  
cd transcoder3  
sudo ./install```  

Inside the transcoder3 directory you will find a large README file and a comprehensive manual in PDF format. The manual is also available [online](http://steinerdatenbank.de/software/rtranscode_manual.pdf)

[Support thread on the Raspberry Pi forum](https://www.raspberrypi.org/forums/viewtopic.php?f=38&t=123876)
