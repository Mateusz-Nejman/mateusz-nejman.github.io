---
layout: post
title: Pixed 1.0 Released
description: Finally I released my latest project.
date: 2024-10-04
categories: [Pixed]
tags: [pixed, pixelart, projects]
image:
  path: /img/2024-10-04-pixed/screenshot1.png
  alt: Pixed screenshot
---

Previously I worked on fork of Piskel app ([Mateusz-Nejman/piskel_touch](https://github.com/Mateusz-Nejman/piskel_touch)), where I made some improvements for better touch screen support. Main problem i saw in this software is to make functionalities both for desktop & web. I used Neutralino.JS for build desktop app, but it was not enough. I decided to port Piskel to C#. But after that I had in my mind "Hey, why I'm porting this app, lets try to make my own software for pixel art!". And finally, I created Pixed, my own pixelart software used by myself and also I hope it will be used by You :smile:

### Features
- Open/save PNG files
- Open Piskel projects
- Open/save own project format
- Improved tools from Pixed
- Touch screen friendly UI (I'm still improving it)

### Stack
- .NET 8
- Avalonia [AvaloniaUI/avalonia](https://github.com/avaloniaui/avalonia)
- Microsoft.Extensions.DependencyInjection [NuGet](https://www.nuget.org/packages/Microsoft.Extensions.DependencyInjection)
- Newtonsoft.JSON [JamesNK/Newtonsoft.Json](https://github.com/JamesNK/Newtonsoft.Json)
- PixiEditor.ColorPicker [PixiEditor/ColorPicker](https://github.com/PixiEditor/ColorPicker)
- System.Reactive (RX) [dotnet/reactive](https://github.com/dotnet/reactive)
- BigGustave [EliotJones/BigGustave](https://github.com/EliotJones/BigGustave)
- LZMA SDK [monemihir/LZMA-SDK](https://github.com/monemihir/LZMASDK)

### Screenshots

![Pixed screenshot1](/img/2024-10-04-pixed/screenshot1.png "Screenshot 1")
![Pixed screenshot2](/img/2024-10-04-pixed/screenshot2.png "Screenshot 2")

### Download

[<img src="https://get.microsoft.com/images/en-us%20dark.svg" alt="Download from Microsoft Store">](https://apps.microsoft.com/detail/9nwzsx6x2bgx?mode=direct)

[Github](https://github.com/Mateusz-Nejman/Pixed)
