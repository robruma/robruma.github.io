---
id: 17
title: Junk Mail Street Cred
date: 2015-06-05T23:26:29+00:00
author: robruma
layout: post
guid: http://invisiblerobots.org/?p=17
permalink: /2015/06/05/junk-mail-street-cred/
tags:
  - tech
---
This complication began when I wanted to setup a Postfix mail system for this site and I figured I didn&#8217;t want to manage any users or mailboxes. I had this idea to only setup an inbound Postfix relay with local-only access. However, I only intended to use aliases pointing to external addresses with actual mailboxes to keep from storing mail on the instances. It was no surprise that I ran into a problem where some mail was getting dropped.
  
It&#8217;s about a 50/50 shot at sending email from an ec2 instance and having it arrive at the intended destination. I&#8217;m pretty sure most mail systems will spam tag an ec2 based message relay unless it originates from the instance. I believe one of the reasons ec2 relayed messages are tagged as spam stems from the remote mail systems client restrictions performing a reverse DNS check on the origin. An AWS instance will always have functional forward and reverse DNS but with a relay through an AWS resource (instance or ELB), the reverse DNS will always be authoritative to Amazon and therefore never resolve to a delegated domain in Route 53. I think there are some hoops you can jump through with Amazon to lift some of the restrictions by using an EIP [yada yada yada&#8230;](https://forums.aws.amazon.com/thread.jspa?messageID=351288) This all seemed like an interesting exercise but I was not thrilled at the notion of using an EIP on a dedicated instance strictly for relaying mail.
  
What I ended up configuring was a seemingly solid email system comprised of Postfix, Postgrey and Amazon SES that I can terminate and rebuild whenever I like. I say &#8220;seemingly&#8221; because it has not been battle tested for very long so far but it has potential to be pretty bulletproof. And, by using CloudFormation and Auto Scaling groups, all these instances essentially become [ephemeral](http://cloudscaling.com/blog/cloud-computing/pets-vs-cattle-the-elastic-cloud-story/).
  
Amazon SES can be used as, basically, a [smart host](http://en.wikipedia.org/wiki/Smart_host) when email is either being relayed or generated from an ec2 instance. Although, you need to verify each email address and/or domain you intend to be delivering to. Here is a straightforward how-to for Postfix:
  
<a href="https://docs.aws.amazon.com/ses/latest/DeveloperGuide/postfix.html" target="_blank">https://docs.aws.amazon.com/ses/latest/DeveloperGuide/postfix.html</a>
  
SES can also set up the necessary DNS entries for SPF and DKIM automatically in Route 53 for the domain in question. If the domain isn&#8217;t delegated to Route 53, SES will allow you to download the proper syntax for the records to be created wherever you&#8217;re hosting DNS.
  
I also didn&#8217;t want to be a bad interweb citizen and a potential spam target so I added a few restrictions and greylisting:
  
<http://wiki.centos.org/HowTos/postfix_restrictions>
  
<http://wiki.centos.org/HowTos/postgrey>
  
For the lazy and the Googlers, here&#8217;s part of the UserData I used to configure the mail hosts:

<pre>yum -y install postfix postfix-perl-scripts postgrey mailx
yum -y remove sendmail

perl -pi -e 's/^inet_interfaces = localhost$/#inet_interfaces = localhost/g' /etc/postfix/main.cf
perl -pi -e 's/^#inet_interfaces = all$/inet_interfaces = all/g' /etc/postfix/main.cf
perl -pi -e 's/^#myhostname = virtual.domain.tld$/myhostname = invisiblerobots.org/g' /etc/postfix/main.cf
perl -pi -e 's/^#mydomain = domain.tld$/mydomain = invisiblerobots.org/g' /etc/postfix/main.cf
perl -pi -e 's/^#myorigin = \$mydomain$/myorigin = \$mydomain/g' /etc/postfix/main.cf
perl -pi -e 's/^mydestination = \$myhostname,\slocalhost.\$mydomain,\slocalhost$/mydestination = \$myhostname, localhost.\$mydomain, localhost, \$mydomain/g' /etc/postfix/main.cf
perl -pi -e 's/^#mynetworks\_style = host$/mynetworks\_style = host/g' /etc/postfix/main.cf
perl -pi -e 's/^\s+-o smtp_fallback_relay=$/#       -o smtp_fallback_relay=/g' /etc/postfix/master.cf

 # Add aliases here
cat &lt;&lt; EOF &gt;&gt; /etc/aliases
someone:        someone@example.org
EOF
newaliases

cat &lt;&lt; EOF &gt;&gt; /etc/postfix/main.cf

 # SES relayhost
relayhost = [${SESEndpoint}]:25

 # Inbound/outbound TLS config
smtpd_use_tls = yes
smtpd_tls_cert_file = /etc/pki/tls/certs/localhost.crt
smtpd_tls_key_file = $smtpd_tls_cert_file
smtp_tls_CAfile = /etc/ssl/certs/ca-bundle.crt
smtpd_tls_received_header = yes
smtpd_tls_session_cache_timeout = 3600s
smtp_sasl_auth_enable = yes
smtp_sasl_security_options = noanonymous
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_use_tls = yes
smtp_tls_security_level = encrypt
smtp_tls_note_starttls_offer = yes
tls_random_source = dev:/dev/urandom

 # SMTP HELO restrictions
smtpd_delay_reject = yes
smtpd_helo_required = yes
smtpd_helo_restrictions =
    permit_mynetworks,
    reject_non_fqdn_helo_hostname,
    reject_invalid_helo_hostname,
    permit

 # SMTP sender restrictions
smtpd_sender_restrictions =
    permit_mynetworks,
    reject_non_fqdn_sender,
    reject_unknown_sender_domain,
    permit

 # SMTP client restrictions
smtpd_recipient_restrictions =
   reject_unauth_pipelining,
   reject_non_fqdn_recipient,
   reject_unknown_recipient_domain,
   permit_mynetworks,
   reject_unauth_destination,
   check_sender_access
         hash:/etc/postfix/sender_access,
   reject_rbl_client zen.spamhaus.org,
   reject_rbl_client bl.spamcop.net,
   check_policy_service unix:postgrey/socket,
   permit
EOF

cat &lt;&lt; EOF &gt; /etc/postfix/sasl_passwd
[${SESEndpoint}]:25 ${SESUser}:${SESPassword}
EOF
postmap hash:/etc/postfix/sasl_passwd
chown root:root /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db
chmod 0600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db

cat &lt;&lt; EOF &gt; /etc/postfix/sender_access
 #
 # Black/Whitelist for senders matching the 'MAIL FROM' field. Examples...
 #
 #myfriend@example.com    OK
 #junk@spam.com           REJECT
 #marketing@              REJECT
 #theboss@                OK
 #deals.marketing.com     REJECT
 #somedomain.com          OK
EOF
postmap hash:/etc/postfix/sender_access
chown root:root /etc/postfix/sender_access /etc/postfix/sender_access.db
chmod 0600 /etc/postfix/sender_access /etc/postfix/sender_access.db

cat &lt;&lt; EOF &gt; /etc/sysconfig/postgrey
OPTIONS="--unix=/var/spool/postfix/postgrey/socket --delay=60"
EOF

service postgrey start
chkconfig postgrey on
service postfix start
chkconfig postfix on</pre>

Update:

As of 9/28/15, AWS [released](https://aws.amazon.com/blogs/aws/new-receive-and-process-incoming-email-with-amazon-ses/) a new feature of SES that allows receiving inbound email which makes this process outdated to a certain extent. It can still be used to send email that originates from an instance to ensure it is free of spam.

Although, you should probably ensure you are not listening for inbound SMTP connections anymore.
  
Remove the following lines from the above example:

<pre>perl -pi -e 's/^inet_interfaces = localhost$/#inet_interfaces = localhost/g' /etc/postfix/main.cf
perl -pi -e 's/^#inet_interfaces = all$/inet_interfaces = all/g' /etc/postfix/main.cf
</pre>
