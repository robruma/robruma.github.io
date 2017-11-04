---
id: 11
title: cfn-pyplates and cfn-init magic
date: 2015-05-25T02:56:59+00:00
author: robruma
layout: post
guid: http://invisiblerobots.org/?p=11
permalink: /2015/05/25/cfn-pyplates-and-cfn-init-magic/
tags:
  - tech
---
For some, this information may be old hat, but for a while I&#8217;ve been looking for a good way to run cfn-init within UserData during an instance launch. I had a catch 22 scenario happening when building UserData with references to AWS::StackId because I&#8217;m using cfn-pyplates to build the CloudFormation json. Until recently, I couldn&#8217;t find a good way to use resource references while base64 encoding a build script in LaunchConfiguration&#8230;
  
The solution turned out to be a combination of `str` and `join` to build the head of the script while reading in the remainder with an `open` call:

<pre>interpreter = str('#!/bin/bash\n')
default_region = join('', str('export AWS_DEFAULT_REGION='), ref('AWS::Region'), str('\n'))
cfn_init_cmd = join('', str('/opt/aws/bin/cfn-init -s '), ref('AWS::StackId'), str(' -r WebLaunchConfiguration --region '), ref('AWS::Region'), str('\n'))
build_script = open('./web_build.sh').read()
user_data_script = join('', interpreter, default_region, cfn_init_cmd, build_script)</pre>

and in my LaunchConfiguration resource:

<pre>cft.resources.add(
  Resource('WebLaunchConfiguration', 'AWS::AutoScaling::LaunchConfiguration',
    {
      'ImageId': ref('InstanceAMI'),
      'InstanceType': ref('EC2InstanceType'),
      'UserData': base64(user_data_script),
      'SecurityGroups': [ref('WebSG'), ref('AdminSG')],
      'KeyName': 'default',
      'IamInstanceProfile': 'WebInstanceRole',
    },
    Metadata(
      {
        'AWS::CloudFormation::Init':
          {
            'config': {
              'files': {
                '/etc/cloud-env.sh': {
                  'content': join('',
                    "export DBName=", ref('DBName'), "\n",
                    "export DBUsername=", ref('DBUsername'), "\n",
                    "export DBPassword=", ref('DBPassword'), "\n",
                    "export DBAddress=", get_att('RDSInstance', 'Endpoint.Address'), "\n",
                    "export DBPort=", get_att('RDSInstance', 'Endpoint.Port'), "\n",
                    "export DBPrefix=", ref('DBPrefix'), "\n",
                ),
                'mode': '000400',
                'owner': 'root',
                'group': 'root'
              },
            }
          }
        }
      }
    )
  )
)</pre>

Boom! Now I could source `/etc/cloud-env.sh` in UserData and have a secure way to pass all my parameters to the instances&#8230;
