---
title: Introducing the Monkey Hi Hat Music Visualizer
tags: c# .net opengl openal visualization graphics
header:
  image: "/assets/headers/monkeyhihat.jpg"
  teaser: "/assets/headers/monkeyhihat.jpg"

# standard header size is 1280 x 400
# image paths:
#   publish:                        (/assets/2023/mm-dd/pic.jpg)
#   edit from _post or _templates:  (../assets/2023/mm-dd/pic.jpg)
---

A customizable audio-reactive music visualizer

<!--more-->

This is just a quick introductory article for my new music visualization program, [Monkey Hi Hat](https://github.com/MV10/monkey-hi-hat). If you're old enough to remember WinAmp and viz plugins like the famous [MilkDrop](https://www.geisswerks.com/about_milkdrop.html), this program is for you. The main difference is that visualizers are relatively easy to create and modify. They are OpenGL-based vertex and fragment shaders, and my [eyecandy](https://github.com/MV10/eyecandy) library intercepts any audio output (we use Spotify) and converts the data to various types of input textures the shaders can read.

The visualizer runs on Windows 10, Windows 11, and has been tested on a Raspberry Pi 4B using the Debian 11 "Bullseye" variant (32-bit).

Additionally, a GUI remote control program called [Monkey Droid](https://github.com/MV10/monkey-droid) is available for Windows and Android devices, allowing you to control Monkey Hi Hat while it runs full-screen on another computer.

Finally, a decent "starter pack" of visualizers is available through my [Volt's Laboratory](https://github.com/MV10/volts-laboratory) repo. Nearly all of these were migrated from [ShaderToy](https://www.shadertoy.com/) or [VertexShaderArt](https://www.vertexshaderart.com/)

## Getting Started

All of these binaries and files can be found on the [releases](https://github.com/MV10/monkey-hi-hat/releases) page of the Monkey Hi Hat repository.

Some configuration steps are required, so be sure to check the Windows and Linux quick-start instructions on the Monkey Hi Hat [wiki](https://github.com/MV10/monkey-hi-hat/wiki).

## More Coming Soon...

I'll be writing several articles about these programs and their constitutent libraries. Many technologies are covered, ranging from OpenAL audio capture, to OpenGL shaders, updates to my own [CommandLineSwitchPipe](https://github.com/MV10/CommandLineSwitchPipe) library, and even the relatively new .NET MAUI cross-platform user interface framework.

Meanwhile, there is a huge amount of information in the README and wiki pages of each of the repositories mentioned above that should keep the technically-curious busy for some time.
