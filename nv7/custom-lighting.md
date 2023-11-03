<!--
title: Crafting a Custom Lighting Solution 
description: Making a custom lighting solution for the Phanteks NV7 using WLED
slug: custom-lighting
published: false
created: 2023-10-02
-->
# Crafting a Custom Lighting Solution 
Ever since I bought my first Phillips Hue RGB lights, I've had an undeniable love for RGB lighting. It wasn't long ago when I worked on embedding custom LEDs for my arcade machine, and oh, the results were worth all the pain! However, as much as I love RGB, when it comes to commercially available PC hardware controllers, it's hard to ignore the limitations that come with them. Many of them just don't offer the range of effects that a true LED enthusiast craves. On the software side, while there are solutions available, they often come with bugs and limiting features, causing more frustration than fascination. And let's not even get started on trying to make different hardware components play nicely together—it's a compatibility nightmare! So, with a mix of passion and a pinch of frustration, I set out to create a custom lighting solution for my new computer case, hoping to overcome these challenges.

## Choosing the hardware
Firstly, all of the LEDs have to be addressable to get the most out of our controller. 

- **The LED controller:** around the rest of the house, I run [WLED](https://kno.wled.ge/) on [QuinLED boards](https://quinled.info/). I love the flexibility and effects WLED gives you. The boards are also small enough to fit in the case and even better, I can run/manage the LEDs independently of the computer which means no more horrible LED software to deal with in Windows.
- **Case lighting:**  the case has already been chosen but we're lucky, the [NV7](https://phanteks.com/NV7.html) comes with some really stylish case lighting.
- **Fans:** this was an easy choice. The [Phanteks D30's](https://phanteks.com/PH-F120D30.html) come with 15 addressable LEDs each, they also come in a reversable variant meaning that no matter where the fan is positioned, we can always get the full RGB effect.
- **Water cooling components:** as well as the classic RGB elements, I wanted to get cooling components with RGB. I settled with the blocks from [EK waterblocks](https://www.ekwb.com/), each component comes with its own addressable RGB strips - even the flow meter!

## Powering the controller
wip

## Parallel vs Series
So if you want to have effects which run up and down a group of LEDs, the LEDs have to be wired in series. The problem is, most PC RGB solutions are wired in parallel. This means your effects will be grouped to each component only and thats not going to give us very exciting effects. So we need to do a bit of custom wiring.

The case comes with 2 led strips which surround the motherboard, wiring these in parallel was easy enough. I soldered a wire to the data terminal on the end of one of the strips and ran that back to the other strip. Now we have 2 strips in series, easy.

The fans are a little more fiddly. They are connected with some proprietry clips with data and power being injected from one side. On the other side, we can use the same clips but need to trim some of the plastic as the clips are keyed to prevent people installing them the wrong way. Now that we have the clips installed backward, its just a case of finding out which wire is connected to the data terminal and wiring that directly to the input data line of the next fan group. Repeat this for each group and now we have 8 fans in series.

The EK water components are where things get tricky. Due to the way the strips are installed into each of the products, there is no way to access the end of each LED strip. This means we cannot extend the data line to the next component and are therefore forced to leave these in parallel. I considered running another microcontroller to sit between all of these components make a virtual series circuit but this seems like a lot of effort for little gain. For now I am happy to have less control over these smaller components and if it bothers me in the future, maybe I'll update it.

# The result
wip
