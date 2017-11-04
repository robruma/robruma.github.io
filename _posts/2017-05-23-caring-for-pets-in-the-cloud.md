---
id: 133
title: Caring for Pets in the Cloud
date: 2017-05-23T10:37:35+00:00
author: robruma
layout: post
guid: https://invisiblerobots.org/?p=133
permalink: /2017/05/23/caring-for-pets-in-the-cloud/
tags:
  - tech
---
Pets vs. Cattle, the widely used <a href="http://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/" target="_blank">phrase</a> forming the analogy between servers and animals is used consistently in the IT industry when automating infrastructure deployments or migrating applications to a cloud vendor.

<img src="https://i0.wp.com/image.slidesharecdn.com/cloudconnect2012-architectures-for-open-and-scalable-clouds-master-rlb-120216195430-phpapp01/95/architectures-for-open-and-scalable-clouds-20-728.jpg?resize=728%2C546&#038;ssl=1" class="aligncenter size-medium" data-recalc-dims="1" />

As System Engineers, Architects and Developers struggle to turn pets into cattle, the underlying application, more than likely, will not be initially architected to take advantage of scalability features. Features like stateless application architecture takes time to become reality and often overrun the time it takes for infrastructure architecture changes. Immutability of servers is not easy either. It starts with how the application is architected. Then, can leverage multiple layers of infrastructure automation, operating system configuration management, acceptance testing, software deployments, continuous integration, etc&#8230;

This may mean expecting extra cost, time and effort when taking a beautifully hand-crafted snowflake application, crossing your fingers and just throwing it into the cloud. Especially if you&#8217;re hoping the <a href="http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-retirement.html" target="_blank">underlying services</a> continue to function at the <a href="https://aws.amazon.com/message/41926/" target="_blank">guaranteed uptime SLA</a> while concurrent efforts to develop cloud native versions or a potential replacement application begin to take shape.

AWS has answered this situation with <a href="https://aws.amazon.com/blogs/aws/new-auto-recovery-for-amazon-ec2/" target="_blank">AutoRecovery</a> which is preferable to an AutoScaling group set to one desired/max instance. Although, an instance stop/start will move your instance to another host, <a href="https://forums.aws.amazon.com/thread.jspa?threadID=72906" target="_blank">AWS does not support live migration</a>. Caveats include the risk of volume corruption and loss of public IP&#8217;s so ensure your instance has an Elastic IP or is behind and ELB and the important data is backed up or <a href="https://aws.amazon.com/answers/infrastructure-management/ebs-snapshot-scheduler/" target="_blank">regular EBS snapshots are taken</a>. 

Google Cloud mitigates instance retirement with <a href="https://cloud.google.com/compute/docs/regions-zones/regions-zones#maintenance" target="_blank">Transparent Maintenance</a> which, as the name suggests, is transparent and requires no additional configuration. By default, instances will &#8220;live migrate&#8221; to healthy hardware but can be configured to &#8220;terminate and reboot&#8221; if the application requires.

Azure doesn&#8217;t seem to have <a href="https://docs.microsoft.com/en-us/azure/virtual-machines/windows/impactful-maintenance" target="_blank">equivalent functionality</a> but has technology in place that enables the underlying virtualization to be updated without a VM reboot, however, will experience a <a href="https://azure.microsoft.com/en-us/updates/azure-in-place-virtual-machine-migration-eliminates-virtual-machine-reboots-during-critical-security-updates-for-host-os/" target="_blank">30 second pause</a>. This does not solve the problem of <a href="https://docs.microsoft.com/en-us/azure/architecture/resiliency/recovery-local-failures" target="_blank">underlying hardware failures</a> which Azure recommends using multiple instances and ensure an <a href="https://azure.microsoft.com/en-us/support/legal/sla/virtual-machines/v1_6/" target="_blank">SLA of 99.9% for single instance deployments</a>.

Be sure to make the right decisions regarding your pets. Research the available options and plan for failure. Turning a kitten into a cow isn&#8217;t easy.

[<img src="https://i2.wp.com/i.imgflip.com/1pkanm.jpg?ssl=1" title="made at imgflip.com" data-recalc-dims="1" />](https://imgflip.com/i/1pkanm)
