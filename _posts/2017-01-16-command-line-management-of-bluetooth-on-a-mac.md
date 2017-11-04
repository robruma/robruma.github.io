---
id: 124
title: Command Line Management of Bluetooth on a Mac
date: 2017-01-16T15:29:16+00:00
author: robruma
layout: post
guid: https://invisiblerobots.org/?p=124
permalink: /2017/01/16/command-line-management-of-bluetooth-on-a-mac/
tags:
  - noob
---
Have you ever been a situation where your good intentions backfire.

For instance, when all you wanted to do was disconnect a Bluetooth headset and end up disconnecting all Bluetooth devices&#8230; Ahem&#8230; Like your mouse&#8230;

I know, it&#8217;s a bonehead move and I had this exact experience because I was in a hurry.

&#8220;Yep, sure, disconnect all Bluetooth devices&#8221; I said to myself as I read the warning out loud&#8230; oops&#8230; this does include the mouse I&#8217;m using at this moment. FFS&#8230;

Ok, now what&#8230; I have no mouse to click the stupid toolbar option to turn Bluetooth back on.

Try the keyboard, nope, no hotkey, not even in System Preferences.

So I start Googling on my phone to try to figure this put and discovered <a href="http://apple.stackexchange.com/questions/47503/how-to-control-bluetooth-wireless-radio-from-the-command-line" target="_blank">blueutil</a>!

This slick little tool put me back in business in under 30 seconds so I thought it was worth a post to give some props.

`brew install blueutil`
  
`blueutil power 1`

Good lookin out <a href="https://github.com/toy/blueutil" target="_blank">Frederik Seiffert</a>!
