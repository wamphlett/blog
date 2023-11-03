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

