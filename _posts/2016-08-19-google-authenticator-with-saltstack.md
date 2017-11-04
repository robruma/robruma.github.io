---
id: 110
title: Google Authenticator with SaltStack
date: 2016-08-19T15:55:54+00:00
author: robruma
layout: post
guid: https://invisiblerobots.org/?p=110
permalink: /2016/08/19/google-authenticator-with-saltstack/
tags:
  - tech
---
I&#8217;ve been working with <a href="https://saltstack.com/" target="_blank">SaltStack</a> for a few weeks now and am becoming a fan. I’ve written a fair amount of both Puppet and Ansible and know it’s not so simple to get started with either.

Puppet’s <a href="https://en.wikipedia.org/wiki/Domain-specific_language" target="_blank">DSL</a> is <a href="https://docs.puppet.com/puppet/latest/reference/lang_summary.html" target="_blank">daunting</a> to some people so the alternative to writing configuration management code defaults to Ansible because of it’s popularity. Some say <a href="http://docs.ansible.com/ansible/" target="_blank">Ansible is easier</a> to understand, but introduces a suspicious <a href="https://www.ansible.com/tower" target="_blank">lock-in</a> factor&#8230; You be the judge.

**My opinion&#8230;**

In a nutshell, compared to Puppet and Ansible, Salt is the best of both worlds without a huge time and financial investment. You get the simplicity of writing configuration management code in YAML yet not bound by a full DSL and have all the power of Python. With that said, I don’t want to get into a full side-by-side-by-side comparison of these three tools. There are plenty of <a href="http://lmgtfy.com/?q=salt+vs+puppet+vs+ansible" target="_blank">comparisons</a> out there to let you decide which is right for the task at hand. I’m just gonna show you how easy it is with Salt.

**Using Augeas to manage PAM**

I’ve jumped on the <a href="https://en.wikipedia.org/wiki/Multi-factor_authentication" target="_blank">MFA</a> band wagon (and you should too) and started using <a href="https://github.com/google/google-authenticator" target="_blank">Google Authenticator</a> for SSH. Atfirst, it seemed like it wasn’t ready for prime time for CentOS. I read a few blog posts about how to set Google Authenticator up and they all used a Debian flavor OS because the PAM module is neatly packaged and available though standard repositories. Not CentOS… What I read used a tarball of the source code, locally compiled and installed using the install method of the Makefile. I really wanted to enable CentOS’ default package management system to handle the PAM module install so I built my own RPM. Which is simpler than it sounds, It’s all detailed <a href="https://github.com/google/google-authenticator/blob/master/libpam/contrib/README.rpm.md" target="_blank">here</a>.

After a successful <a href="http://www.rpm.org/max-rpm-snapshot/rpmbuild.8.html" target="_blank">rpmbuild</a>, I had a shiny new RPM package, with the possibility of being managed by yum, that was installable without having to install compilation tools which having available pose a security risk.

Using Salt, I was able to use the built-in package management <a href="https://docs.saltstack.com/en/latest/ref/states/all/salt.states.pkg.html" target="_blank">state module</a> to install the PAM module. Win!

**Now off to configuration&#8230;**

I’m not gonna detail how to setup a Salt master environment here, there are plenty of <a href="https://docs.saltstack.com/en/latest/topics/tutorials/walkthrough.html" target="_blank">resources</a> out there to help you with all the different options that are right for you. Let’s just assume you have a master and are able to orchestrate the <a href="https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.state.html" target="_blank">highstate</a> on your minions.

Augeas is pretty much the de facto standard in most configuration management tools to handle updates to any text based configuration files. Not surprising to see Salt is embracing this method and look forward to full <a href="https://docs.saltstack.com/en/latest/ref/states/all/salt.states.augeas.html" target="_blank">support</a>.

The code below should modify `/etc/pam.d/sshd` to insert the `pam_google_authenticator.so` module line after the `pam_sepermit.so` line and remove the `password-auth` line since I don’t use password authentication. You may want to consider leaving the `password-auth` line if you actually use password authentication.

<pre>pam_ins_google_authenticator:
  augeas.change:
    - context: /files/etc/pam.d/sshd
    - changes:
      - ins 999 after "*[type = 'auth'][control = 'required'][module = 'pam_sepermit.so']"
      - set 999/type auth
      - set 999/control required
      - set 999/module pam_google_authenticator.so
    - unless: grep -Eq '^auth[[:blank:]]*required[[:blank:]]*pam_google_authenticator.so$' /etc/pam.d/sshd

pam_rm_password-auth:
  augeas.change:
    - context: /files/etc/pam.d/sshd
    - changes:
      - rm "*[type = 'auth'][control = 'substack'][module = 'password-auth']"
    - onlyif: grep -Eq '^auth[[:blank:]]*substack[[:blank:]]*password-auth$' /etc/pam.d/sshd
</pre>

I also needed to make a few adjustments to `/etc/ssh/sshd_config`

The code below should insert (or update) two lines in `/etc/ssh/sshd_config` that control SSH support for keyboard-interactive authentication. You need this to allow sshd to prompt for the validation code needed by the Google Authenticator service. Again, I use publickey authentication and not password so you may need to adjust the `AuthenticationMethods` to your environment.

<pre>sshd_config_enable_ChallengeResponse:
  augeas.change:
    - context: /files/etc/ssh/sshd_config
    - changes:
      - set ChallengeResponseAuthentication yes
    - unless: grep -Eq '^ChallengeResponseAuthentication yes$' /etc/ssh/sshd_config
    - watch_in:
      - service: sshd

sshd_config_enable_AuthenticationMethods:
  augeas.change:
    - context: /files/etc/ssh/sshd_config
    - changes:
      - set AuthenticationMethods publickey,keyboard-interactive
    - unless: grep -Eq '^AuthenticationMethods publickey,keyboard-interactive$' /etc/ssh/sshd_config
    - watch_in:
      - service: sshd
</pre>

Now run salt to apply the highstate to the minion. Assuming you’ve created additional states to manage the package install, this should configure PAM and sshd properly.

**
  
PAM and sshd is configured&#8230; Now what?
  
** 

Fantastic! You just need to run the configuration of an MFA for any local user using this command:

`google-authenticator -tdf --rate-limit=3 --rate-time=30 --window-size=17`

And scan the QR code.

The resulting `~/.google_authenticator` file can be managed using <a href="https://docs.saltstack.com/en/latest/ref/states/all/salt.states.file.html" target="_blank">Salt</a> but should be treated as sensitive information stored on disk.
