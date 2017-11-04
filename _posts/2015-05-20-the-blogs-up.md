---
id: 6
title: 'The blog&#8217;s up!'
date: 2015-05-20T02:10:26+00:00
author: robruma
layout: post
guid: http://invisiblerobots.org/?p=6
permalink: /2015/05/20/the-blogs-up/
tags:
  - newb
---
Finally! Hello World! It&#8217;s about time something made it to the interwebs from this site&#8230; Even though it still looks pretty bare, I&#8217;m kinda digging the clean look of this CMS. You can find it on <a href="https://github.com/anchorcms/anchor-cms">https://github.com/anchorcms/anchor-cms</a> if you&#8217;re interested.

For the record, I&#8217;m beginning somewhat of a journey (late as it may be) of working with cloud enabled services for my employer. My group has been tasked with moving a shitload of applications (600+) into the ever-changing cloud. Some homegrown Java and Python apps and some prepackaged and licensed by those huge corporations (you know the ones I&#8217;m talking about). I figure I&#8217;m gonna learn a lot and make a ton of mistakes. What better way to immortalize my successes and failures than to share with other nerds like me.

I&#8217;m going to try to document as many adventures I come across regarding tools, automation and technologies used in cloud services and whatever interesting past, present and future experiences. I&#8217;m aiming at being as platform agnostic as possible since I may not have a choice about the application that gets to move next. Hopefully I can come up with some good knowledge to share but first, I decided to start right here with this site, which is fully operational in AWS.

I&#8217;m woking on getting an SSL cert for this site thats actually trusted but for now I&#8217;m gonna get by with a busted self-signed cert. I came across a few services that offer free SSL but as you might imagine, they all have advantages and drawbacks.

Here are a few I&#8217;m considering:

<a href="http://www.cacert.org/">http://www.cacert.org/</a>

<a href="https://www.startssl.com/">https://www.startssl.com/</a>

<a href="https://www.instantssl.com/free-ssl-certificate.html">https://www.instantssl.com/free-ssl-certificate.html</a>

Some of these only give you a 30 or 90 day trial certificate which is a good start but Im looking for something more permanent considering the frugality this site will be shooting for&#8230; $0

I also ran across this interesting project with appears to be quite promising:

<a href="https://letsencrypt.org/">https://letsencrypt.org/</a>

It&#8217;s backed by some big name sponsors like the Linux Foundation and promises to be open. It&#8217;s not officially available until sometime in the middle of 2015 but it&#8217;s on the right track.

That&#8217;s all I got for now&#8230;
