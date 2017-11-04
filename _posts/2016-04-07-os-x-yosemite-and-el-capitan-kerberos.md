---
id: 91
title: OS X Yosemite and El Capitan Kerberos
date: 2016-04-07T22:05:17+00:00
author: robruma
layout: post
guid: http://invisiblerobots.org/?p=91
permalink: /2016/04/07/os-x-yosemite-and-el-capitan-kerberos/
tags:
  - tech
---
At one point in time, I used MIT Kerberos extensively throughout my workplace environment. LDAP was extended to use GSS-API and was extremely useful until this:

<pre>$ kinit me@INVISIBLEROBOTS.ORG
me@INVISIBLEROBOTS.ORG's Password:
kinit: krb5_get_init_creds: Preauth required but no preauth options send by KDC
$

</pre>

The above started happening after I upgraded to OS X Yosemite where Apple basically stopped supporting weak DES and RC4 encryption types on their Kerberos implementation shipped on 10.10 and future OS versions; <a href="https://tools.ietf.org/id/draft-ietf-krb-wg-des-die-die-die-04.html" target="_blank">and rightfully so.</a>

I love this part&#8230;

> By 2008, commercial hardware costing less than USD 15,000 could break DES keys in less than a day on average. DES is long past its sell-by date.

You can probably do it these days for about <a href="http://aws.amazon.com/ec2/pricing/" target="_blank">$2.60/hour</a>

To get back the functionality on Yosemite without the necessary update on the KDC side (<a href="http://web.mit.edu/kerberos/krb5-1.12/doc/admin/conf_files/kdc_conf.html#encryption-types" target="_blank">however recommended</a>), follow these steps:

  1. Install <a href="http://brew.sh/" target="_blank">Homebrew</a>
  2. `brew install Caskroom/cask/xquartz`
  3. `brew install homebrew/dupes/heimdal`

You should now have your ability to kinit back&#8230; for now&#8230;

<pre>$ /usr/local/Cellar/heimdal/1.6rc2_1/bin/kinit me@INVISIBLEROBOTS.ORG
me@INVISIBLEROBOTS.ORG's Password:
$ klist
Credentials cache: API:A5DE7730-A162-40ED-B44A-643C6B962C6F
Principal: me@INVISIBLEROBOTS.ORG

Issued Expires Principal
Apr 7 21:01:05 2016 Apr 8 07:01:05 2016 krbtgt/INVISIBLEROBOTS.ORG@INVISIBLEROBOTS.ORG
$
</pre>
