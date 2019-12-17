---
title: Capturing A/V Multimedia Streams From the Browser
author: jgoulah
type: post
date: 2013-06-12T02:29:13+00:00
url: /2013/06/capturing-av-mutimedia-streams-in-browser/
categories:
  - Coding
  - Real-time Web
tags:
  - audio
  - getUserMedia
  - MediaStreamRecorder
  - video
  - webm

---
## Background

I&#8217;ve recently been working on finishing up an application to help gather presentation feedback when demo&#8217;ing a talk for the first time in front of a live audience. The app will play back a video stream and overlay the audiences comments on top of the video in real time. However I haven&#8217;t open sourced it yet because there was a dependency on launching an external video application to do the recording, and have the user upload that to the server. I wanted the application to do &#8220;in browser&#8221; audio/video recording so that it was more seamless with the overall web application. 

Turns out A/V recording is still pretty experimental on the web, but is slowly rolling out in major web browsers. The key component to understand when dealing with A/V capture is the <a href="http://dev.w3.org/2011/webrtc/editor/getusermedia.html" target="_blank">getUserMedia API</a>. The getUserMedia API is an in progress API to access audio and video streams. You can find a decent <a href="http://www.html5rocks.com/en/tutorials/getusermedia/intro/" target="_blank">intro</a> on this API here. 

I was then wondering how to capture the streams recorded by the API and found <a href="http://ericbidelman.tumblr.com/post/31486670538/creating-webm-video-from-getusermedia" target="_blank">this article</a> which covers video only recording. The <a href="http://dev.w3.org/2011/webrtc/editor/webrtc-20111004.html#mediastreamrecorder" target="_blank">MediaStreamRecorder API</a> is currently <a href="https://code.google.com/p/chromium/issues/detail?id=113676" target="_blank">unimplemented</a> in chrome, but there is a nice interface implemented in <a href="https://github.com/antimatter15/whammy" target="_blank">whammy.js</a> that the author uses to demonstrate recording video. However, we also need sound, and it turns out based on the comments in that article that others are looking for that too. 

There is another github project that I found called <a href="https://github.com/mattdiamond/Recorderjs" target="_blank">recorder.js</a> which can help us record the audio stream. It uses the <a href="https://dvcs.w3.org/hg/audio/raw-file/tip/webaudio/specification.html#AudioContext-section" target="_blank">AudioContext</a> interface to export the audio as a blob. 

Combining these two ideas, I can capture separate audio and video streams, and so they must be combined. Theoretically this could be done with a javascript encoder, but I wasn&#8217;t sure if one exists that fits this use case or how much computing power that would take on the client side. Therefore I chose to encode the streams server side with <a href="http://libav.org/avconv.html" target="_blank">avconv</a> (aka. ffmpeg). When the recording is completed the client sends HTTP POST requests to the server with the content of each stream. The video is <a href="http://www.webmproject.org/" target="_blank">webm</a> format the and audio is <a href="http://en.wikipedia.org/wiki/WAV" target="_blank">wav</a>.

The command to do our transcoding is pretty simple. On the server side after you have received the files from our POST requests, you would do something like this, replacing the input files names with those from the recording:

{{< highlight bash >}}avconv -i input.wav -i input.webm -acodec copy -vcodec copy output.mkv{{< /highlight >}}

And the file _output.mkv_ that is produced will contain the video with your audio stream. 

## Implementation

If you&#8217;re interested in an example implementation you can find <a href="https://github.com/jgoulah/webrecorder" target="_blank">my code here</a>. My next steps are to see if I can dig deeper into <a href="https://sites.google.com/site/webrtc/" target="_blank">WebRTC</a> to send both streams to my server and transcode them on the fly, instead of consuming the resources on the client side to hold the audio and video streams in memory until the video is complete. If you have a better way of accomplishing A/V recording please let me know in the comments!
