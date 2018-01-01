---
title: hello_world.asm
tags: personal
header:
  image: "/assets/2017/12-22/helloworld.png"
  teaser: "/assets/2017/12-22/helloworld.png"
---


"Everybody should blog," said nobody, ever. And yet here we are. Posts will focus on software development, particularly C#, .NET, Azure, and Unity games, with the occasional foray into weird stuff I'm doing at the office like IBM ODM business rules (I can't believe some people actually like using Eclipse). I suppose I should add web development to that list. I'm not especially fond of web development -- I consider HTML, CSS, and Javascript three trainwrecks that just never seem to end -- but it pays the bills better than most things I can do from the comfort of an air-conditioned office. Non-software topics are bound to show up, too. Electronics, AI, old cars, and motorsports come to mind, although I've been out of the racing loop for a few years. Heck, maybe even beer, guns, and small dogs. You never know.

<!--more-->

## http:<i></i>//about:jonmcguire

This post is a test-run for hosting a blog using a GitHub "personal page" site. This approach to content-management was inspired by [Marc Duiker's post](https://blog.marcduiker.nl/2015/10/06/moving-my-blog-i-love-github-and-markdown.html) detailing how he streamlined his own blogging process, thereby avoiding the headaches of owning and operating a real CMS. Avoiding HTML is just icing on the cake. I'm not overly fond of this template, but it works well enough for the moment. Did I mention I hate CSS?

So what's this "mcguirev10" thing all about? For nearly 20 years, it has (mostly) been my username, sometimes abbreviated to "MV10". We bought a Viper in 2001 and kept that for 16 years, so most people assume it's *that* V10, but the reality is my mudding days preceded my track days. Back in October of 1999, I decided towing our big boat with a Dodge Durango was no fun, so I placed an order for a Dodge Ram 2500 with a V10. The Facebook guys were still in high school, and discussion forums were the social media of the day. I had a question about the truck, but someone else already had the username "mcguire" on the truck owner's forum, so I added the V10 and the rest is history.

![going racing](/assets/2017/12-22/goneracing.jpg)

Anyway, as I mentioned, this post is really a test for sorting out the templates and styles and other browser-mandated arcana, but since I'm making all this noise about code, here's some 8086 assembly to spice things up before we leave.

```nasm
DATA SEGMENT
     MSG DB "hello, world$"
ENDS
CODE SEGMENT  
    ASSUME DS:DATA CS:CODE
START:
      MOV AX,DATA
      MOV DS,AX
      MOV DX,OFFSET MSG       
      MOV AH,9H
      INT 21H
      MOV AH,4CH
      INT 21H      
END START
ENDS
```
