---
id: 169
title: Finding aws-cfn-bootstrap hidden SELinux violations
date: 2017-06-24T14:16:10+00:00
author: robruma
layout: post
guid: https://invisiblerobots.org/?p=169
permalink: /2017/06/24/finding-aws-cfn-bootstrap-hidden-selinux-violations/
tags:
  - tech
---
Using <a href="http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-helper-scripts-reference.html" target="_blank">aws-cfn-bootstrap</a> helper scripts on Amazon Linux is really simple, straightforward and it comes pre-installed. However, using the helper scripts on a Red Hat variant has it&#8217;s challenges.

One such challenge is installation of the helper scripts on RHEL/CentOS 7. Trying to install the <a href="https://s3.amazonaws.com/cloudformation-examples/aws-cfn-bootstrap-latest.amzn1.noarch.rpm" target="_blank">RPM</a> does not work out of the box. Running one of the scripts yields:

<pre># ./cfn-init 
Traceback (most recent call last):
  File "./cfn-init", line 19, in &lt;module>
    import cfnbootstrap
ImportError: No module named cfnbootstrap
</pre>

I have not found a way to install the missing Python module directly from pip but installing the helper scripts from the provided tarball works much better:

<pre>/usr/bin/easy_install --script-dir /opt/aws/bin https://s3.amazonaws.com/cloudformation-examples/aws-cfn-bootstrap-latest.tar.gz
</pre>

Now, I believe I have a working installation of the helper scripts&#8230; Until I try attempting the next challenge&#8230; Configuring a hook that executes a command other than the standard `/opt/aws/bin/cfn-init --stack ${AWS::StackId} --resource EC2Instance --region ${AWS::Region}` command which is intended to re-execute the user-data on changes to the stack resource&#8217;s metadata:

<pre>/etc/cfn/hooks.d/cfn-user-data.conf:
  content:
    Fn::Sub: |
      [cfn-user-data-hook]
      triggers=post.update
      path=Resources.EC2Instance.Metadata.AWS::CloudFormation::Init
      action=/bin/curl -sf http://169.254.169.254/latest/user-data | bash -x
      runas=root
  mode: '000400'
  owner: root
  group: root
</pre>

So&#8230; I run some more tests and determine the user-data was not re-executed as I intended.

I start troubleshooting and there&#8217;s nothing obvious in logs. Which is confusing because running cfn-init as a one-shot command creates files described in metadata&#8230; WTH is going on.

Finally, I realize SELinux is in enforcing mode. I run `setenforce 0` and magically, all is well.

The simplest fix would be to run a command to <a href="https://stopdisablingselinux.com/" target="_blank">permanently disable SELinux</a> system wide and put the whole problem behind me, but this way of thinking has the potential of causing countless other problems&#8230;

So, I decided to dig in and figure out why SELinux is preventing the execution of commands other than cfn-init.

First thought &#8212; setroubleshoot &#8212; Yes, this should be simple&#8230; 

I re-enable SELinux, install setroubleshoot and run tests once again and the failure returns.

But when I try to find the violation in the audit logs, I get this:

<pre># sealert -a /var/log/audit/audit.log 
100% done
found 0 alerts in /var/log/audit/audit.log
</pre>

Whaaaa??

I actually have never run into this before&#8230; Typically, SELinux is good about logging reasons why commands and processes are denied execution. So now I&#8217;m in un-charted territory. Until a lengthy Google-ing session uncovered this process to <a href="https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Security-Enhanced_Linux/sect-Security-Enhanced_Linux-Fixing_Problems-Possible_Causes_of_Silent_Denials.html" target="_blank">disable dontaudit rules</a>.

I execute:

<pre>semodule -DB
</pre>

And re-execute some test changes through a stack update and I start seeing some violations in the audit log:

<pre>--------------------------------------------------------------------------------

SELinux is preventing /usr/bin/bash from using the rlimitinh access on a process.

*****  Plugin catchall (100. confidence) suggests   **************************

If you believe that bash should be allowed rlimitinh access on processes labeled initrc_t by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c '00-netreport' --raw | audit2allow -M my-00netreport
# semodule -i my-00netreport.pp

--------------------------------------------------------------------------------

SELinux is preventing /usr/bin/python2.7 from using the rlimitinh access on a process.

*****  Plugin catchall (100. confidence) suggests   **************************

If you believe that python2.7 should be allowed rlimitinh access on processes labeled setroubleshootd_t by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'setroubleshootd' --raw | audit2allow -M my-setroubleshootd
# semodule -i my-setroubleshootd.pp
</pre>

Ok, this is something but still not very useful&#8230; The suggestions to allow the access do not directly reference any of the commands I&#8217;m running so I&#8217;m skeptical that creating the policies to allow &#8220;rlimitinh access on processes&#8221; will work.

I found a <a href="https://bugzilla.redhat.com/show_bug.cgi?id=1183659" target="_blank">bug report</a> of a similar error which gave me the idea that I may have a filesystem label problem. Ok, easy enough, I run this:

<pre># restorecon -R -v /usr
restorecon reset /usr/lib/python2.7/site-packages/aws_cfn_bootstrap-1.4-py2.7.egg/EGG-INFO/scripts context system_u:object_r:lib_t:s0->system_u:object_r:bin_t:s0
restorecon reset /usr/lib/python2.7/site-packages/aws_cfn_bootstrap-1.4-py2.7.egg/EGG-INFO/scripts/cfn-init context system_u:object_r:lib_t:s0->system_u:object_r:bin_t:s0
restorecon reset /usr/lib/python2.7/site-packages/aws_cfn_bootstrap-1.4-py2.7.egg/EGG-INFO/scripts/cfn-signal context system_u:object_r:lib_t:s0->system_u:object_r:bin_t:s0
restorecon reset /usr/lib/python2.7/site-packages/aws_cfn_bootstrap-1.4-py2.7.egg/EGG-INFO/scripts/cfn-get-metadata context system_u:object_r:lib_t:s0->system_u:object_r:bin_t:s0
restorecon reset /usr/lib/python2.7/site-packages/aws_cfn_bootstrap-1.4-py2.7.egg/EGG-INFO/scripts/cfn-hup context system_u:object_r:lib_t:s0->system_u:object_r:bin_t:s0
restorecon reset /usr/lib/python2.7/site-packages/aws_cfn_bootstrap-1.4-py2.7.egg/EGG-INFO/scripts/cfn-elect-cmd-leader context system_u:object_r:lib_t:s0->system_u:object_r:bin_t:s0
restorecon reset /usr/lib/python2.7/site-packages/aws_cfn_bootstrap-1.4-py2.7.egg/EGG-INFO/scripts/cfn-send-cmd-result context system_u:object_r:lib_t:s0->system_u:object_r:bin_t:s0
restorecon reset /usr/lib/python2.7/site-packages/aws_cfn_bootstrap-1.4-py2.7.egg/EGG-INFO/scripts/cfn-send-cmd-event context system_u:object_r:lib_t:s0->system_u:object_r:bin_t:s0
</pre>

Well, ain&#8217;t that something, I relabeled all the files installed by the aws-cfn-bootstrap tarball.

The commands now execute properly so I add this relabel command as part of the install process for aws-cfn-bootstrap.

PS, if you&#8217;ve followed along so far, don&#8217;t forget to re-enable the dontaudit rules:

<pre>semodule -B
</pre>

So the audit logs don&#8217;t fill up with garbage.
