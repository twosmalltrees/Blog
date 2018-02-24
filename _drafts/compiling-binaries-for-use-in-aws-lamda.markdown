---
layout: default
title:  "Compiling Binaries For Use In AWS Lambda"
description: "Running arbitrary exectuables in AWS Lambda brings a huge range of possibilities in reach of serverless architectures. In this post I walk through a general procedure to compile binaries for use in AWS Lambda, using ffmpeg as an example."
date:   2017-12-06 08:28:24 +1100
readableDate: "December 6th, 2017"
categories: serverless
---

* Initialise EC2 instance (micro is fine) 
* Create new Key
* In terminal, navigate to location of Key
* SSH into terminal

* Install dependencies
* wget ffmpeg
* configure build
* make
* scp the binary onto your machine

In case you've missed the hype train, Lambda Functions are pretty neat. I won't go on about *why* you should be look into in Lambda and serverless architectures, as it's been covered pretty extensively elsewhere, and I don't really have anything new to add. Do a google if you need a refresher.

Anyhow, I recently found myself looking into how/if it might be possible to build a credit roll detector for video content using Lambda. I haven't quite got that one figured out yet (there'll be a blog post if I ever get something half decent up and running), however it quickly became clear that I'd need to do some initial processing with [ffmpeg][ffmpeg], to extract individual frames from the source video. It also quickly became clear that I had no idea how or if you could use a native libraries like ffmpeg in Lambda. 

Turns out it is

### Step 1: Get you an EC2




Useful sources
-------------

https://trac.ffmpeg.org/wiki/CompilationGuide
https://aws.amazon.com/blogs/compute/nodejs-packages-in-lambda/
https://netninja.com/2017/04/30/building-lambda-binaries-in-the-cloud/
And related https://github.com/BrianEnigma/StaticBinaries

[ffmpeg]: https://www.ffmpeg.org
[serverless]: https://serverless.com/
