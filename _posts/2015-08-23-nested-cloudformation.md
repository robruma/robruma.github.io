---
id: 38
title: Nested CloudFormation
date: 2015-08-23T00:14:00+00:00
author: robruma
layout: post
guid: https://invisiblerobots.org/?p=38
permalink: /2015/08/23/nested-cloudformation/
categories:
  - tech
---
It&#8217;s been a while since the last post, partially because I&#8217;m lazy. But also, life has been busy. I switched the site to a new <a href="https://github.com/robruma/ansible-wordpress" target="_blank">CMS</a> and <a href="https://theme.wordpress.com/themes/satellite/" target="_blank">theme</a> and a new way to <a href="http://www.google.com/adsense" target="_blank">pay the bills.</a> I&#8217;ve realized there&#8217;s no way to sustain this site effectively without some way to pay for the AWS resources once the free tier year is over. So, click some ads so I can keep it running!

Anyway, one of the changes you may not have noticed is the use of <a href="http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-stack.html" target="_blank">nested CloudFormation stacks</a> to build the site&#8217;s necessary resources in AWS. Nested stacks are basically a way to _&#8220;resourceify&#8221;_ an entire stack so other stacks can use, or <a href="https://en.wikipedia.org/wiki/Reusability" target="_blank">reuse</a> the output attribute information as parameter values to other stacks. This allows you to &#8220;chain&#8221; stacks together that are <a href="http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-dependson.html" target="_blank">dependent</a> on one another so you can modularize the function of the stack. You can have a base-level resource stack that creates all the necessary security groups, the ELB and any Route 53 records which <a href="http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/outputs-section-structure.html" target="_blank">outputs</a> the resource names. The next stack, say a stack that creates an RDS, can use these outputs with a <a href="http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-getatt.html" target="_blank"><code>Fn::GetAtt</code> function</a> as parameter values. And so on&#8230;

&#8220;Can&#8217;t you just create one gigantic stack so you don&#8217;t need to go though all this horseshit&#8221;, you ask?

I&#8217;m glad you asked! Yeah, you can write up one monolithic stack, within <a href="http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html" target="_blank">limits</a>, that creates all the resources you need and orders the dependencies correctly. However, what happens when you want to change or update a resource? e.g. Change a password or modify a security group or utilize a new feature of a resource. You can certainly parameterize the stack to make it easier, but issuing an `<a href="http://docs.aws.amazon.com/cli/latest/reference/cloudformation/update-stack.html" target="_blank">update-stack</a>` on a stack sometimes has unforeseen effects, like inadvertently recreating an RDS because you updated a property that required <a href="http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks.html" target="_blank">replacement</a>! Yes, this happens.

Updating a top-level stack to change a parameter in a nested stack only updates the stack where you made the change. For instance, I added a new IP to a security group that is applied to the EC2 instances. I updated the top-level stack which updated the base-level stack only, guaranteeing the RDS stack was unaltered. In a sense, this adheres to the <a href="https://en.wikipedia.org/wiki/Principle_of_least_privilege" target="_blank">Principle of Least Privilege</a> by only updating the necessary stacks. Ok, that&#8217;s a stretch, but you get my point.

Also, there is value in creating reusable stacks that can be shared among many sites. For instance, a parameterized RDS stack can be hosted in an S3 bucket and reused repeatedly in multiple top-level stacks.

Now imagine this hypothetical situation. Say you manage a large number of applications that all use a MySQL RDS. Then OMG, out of the blue, you read all about a new, nasty security vulnerability with the version of MySQL used on all the applications. Luckily, there is an updated version available but uh oh, it&#8217;s a major release. You rush to test the new version and verify all apps can be upgraded. Then you realize you set `AllowMajorVersionUpgrade` to `false` on all RDS instances built with the stack. Thankfully, you had the foresight to reuse this stack as a nested stack resource so updating this parameter is cake.

The alternative may have been updating each stack individually while cursing like a drunken sailor the entire time. No offense intended to any overindulging cloud engineers with a colorful vocabulary who happen to be working on a boat. It&#8217;s pure coincidence.

Examples:

**Warning, these <del datetime="2015-08-23T04:00:04+00:00">may</del> will not work as-is**

top-level</p> 

<pre>{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "CloudFormation top-level stack",
  "Parameters": {
    "SSLCertificateId": {
      "Type": "String",
      "Description": "The SSL certificate used on the ELB"
    },
    "HostedZoneName": {
      "Type": "String",
      "Description": "Name of the hosted zone in Route 53"
    },
    "KeyName": {
      "Type": "String",
      "Description": "Name of the SSH Key used to access the instances"
    },
    "EC2InstanceType": {
      "ConstraintDescription": "Must use a valid instance type",
      "Type": "String",
      "Description": "Instance type to use for created Server EC2 instance"
    },
    "EC2InstanceName": {
      "Type": "String",
      "Description": "Instance name tag for the EC2 instance"
    },
    "AutoScalingDesiredCapacity": {
      "Type": "String",
      "Description": "Auto Scaling Group desired capacity"
    },
    "AutoScalingMinSize": {
      "Type": "String",
      "Description": "Auto Scaling Group minimum size"
    },
    "AutoScalingMaxSize": {
      "Type": "String",
      "Description": "Auto Scaling Group maximum size"
    },
    "InstanceAMI": {
      "ConstraintDescription": "Must be a valid AMI",
      "Type": "String",
      "Description": "AMI ID"
    },
    "RDSEngine": {
      "Type": "String",
      "Description": "The RDS engine type"
    },
    "RDSEngineVersion": {
      "Type": "String",
      "Description": "The RDS engine version"
    },
    "RDSDBName": {
      "AllowedPattern": "[a-zA-Z][a-zA-Z0-9]*",
      "ConstraintDescription": "Must begin with a letter and contain only alphanumeric characters",
      "Description": "The RDS name",
      "MaxLength": "64",
      "MinLength": "1",
      "Type": "String"
    },
    "RDSUsername": {
      "AllowedPattern": "[a-zA-Z][a-zA-Z0-9]*",
      "ConstraintDescription": "Must begin with a letter and contain only alphanumeric characters",
      "Description": "The RDS admin account username",
      "MaxLength": "16",
      "MinLength": "1",
      "Type": "String"
    },
    "RDSPassword": {
      "ConstraintDescription": "Must contain only alphanumeric characters",
      "Description": "The RDS admin account password",
      "MinLength": "1",
      "AllowedPattern": "[a-zA-Z0-9]*",
      "NoEcho": "true",
      "MaxLength": "40",
      "Type": "String"
    },
    "RDSDBPrefix": {
      "Type": "String",
      "Description": "The RDS table name prefix"
    },
    "RDSDBInstanceClass": {
      "Type": "String",
      "Description": "The RDS instance class"
    },
    "RDSAllocatedStorage": {
      "Type": "String",
      "Description": "The size of the storage allocated to the RDS"
    },
    "RDSAllowMajorVersionUpgrade": {
      "Type": "String",
      "Description": "Whether to allow major version upgrades to the RDS",
      "AllowedValues": [
        "true",
        "false"
      ]
    },
    "RDSAutoMinorVersionUpgrade": {
      "Type": "String",
      "Description": "Whether to allow automatic minor version upgrades to the RDS",
      "AllowedValues": [
        "true",
        "false"
      ]
    },
    "RDSPort": {
      "Type": "String",
      "Description": "The RDS port"
    },
    "BaseStackTemplateURL": {
      "Type": "String",
      "Description": "The Base Stack template URL"
    },
    "RDSStackTemplateURL": {
      "Type": "String",
      "Description": "The RDS Stack template URL"
    },
    "EC2StackTemplateURL": {
      "Type": "String",
      "Description": "The EC2 Stack template URL"
    }
  },
  "Resources": {
    "BaseStack": {
      "Type": "AWS::CloudFormation::Stack",
      "Properties": {
        "TemplateURL": {
          "Ref": "BaseStackTemplateURL"
        },
        "Parameters": {
          "HostedZoneName": {
            "Ref": "HostedZoneName"
          },
          "SSLCertificateId": {
            "Ref": "SSLCertificateId"
          }
        }
      }
    },
    "RDSStack": {
      "Type": "AWS::CloudFormation::Stack",
      "Properties": {
        "TemplateURL": {
          "Ref": "RDSStackTemplateURL"
        },
        "Parameters": {
          "RDSAllocatedStorage": {
            "Ref": "RDSAllocatedStorage"
          },
          "VPCSecurityGroup": {
            "Fn::GetAtt": [
              "BaseStack",
              "Outputs.RDSSGName"
            ]
          },
          "RDSUsername": {
            "Ref": "RDSUsername"
          },
          "RDSPassword": {
            "Ref": "RDSPassword"
          },
          "RDSDBName": {
            "Ref": "RDSDBName"
          },
          "RDSDBInstanceClass": {
            "Ref": "RDSDBInstanceClass"
          },
          "RDSAllowMajorVersionUpgrade": {
            "Ref": "RDSAllowMajorVersionUpgrade"
          },
          "RDSAutoMinorVersionUpgrade": {
            "Ref": "RDSAutoMinorVersionUpgrade"
          }
        }
      },
      "DependsOn": "BaseStack"
    },
    "EC2Stack": {
      "Type": "AWS::CloudFormation::Stack",
      "Properties": {
        "TemplateURL": {
          "Ref": "EC2StackTemplateURL"
        },
        "Parameters": {
          "WebInstanceProfile": {
            "Fn::GetAtt": [
              "BaseStack",
              "Outputs.WebInstanceProfileName"
            ]
          },
          "RDSUsername": {
            "Ref": "RDSUsername"
          },
          "RDSEndpointAddress": {
            "Fn::GetAtt": [
              "RDSStack",
              "Outputs.RDSEndpointAddress"
            ]
          },
          "Environment": {
            "Ref": "Environment"
          },
          "RDSPassword": {
            "Ref": "RDSPassword"
          },
          "SSLCertificateId": {
            "Ref": "SSLCertificateId"
          },
          "RDSDBName": {
            "Ref": "RDSDBName"
          },
          "AutoScalingMaxSize": {
            "Ref": "AutoScalingMaxSize"
          },
          "HostedZoneName": {
            "Ref": "HostedZoneName"
          },
          "RDSEndpointPort": {
            "Fn::GetAtt": [
              "RDSStack",
              "Outputs.RDSEndpointPort"
            ]
          },
          "KeyName": {
            "Ref": "KeyName"
          },
          "InstanceAMI": {
            "Ref": "InstanceAMI"
          },
          "AutoScalingMinSize": {
            "Ref": "AutoScalingMinSize"
          },
          "WebELB": {
            "Fn::GetAtt": [
              "BaseStack",
              "Outputs.WebELBName"
            ]
          },
          "S3Bucket": {
            "Fn::GetAtt": [
              "BaseStack",
              "Outputs.S3BucketName"
            ]
          },
          "EC2InstanceName": {
            "Ref": "EC2InstanceName"
          },
          "EC2InstanceType": {
            "Ref": "EC2InstanceType"
          },
          "AutoScalingDesiredCapacity": {
            "Ref": "AutoScalingDesiredCapacity"
          },
          "WebSG": {
            "Fn::GetAtt": [
              "BaseStack",
              "Outputs.WebSGName"
            ]
          }
        }
      },
      "DependsOn": "RDSStack"
    }
  }
}
</pre>

base</p> 

<pre>{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "Base Resources stack",
  "Parameters": {
    "SSLCertificateId": {
      "Type": "String",
      "Description": "The SSL certificate used on the ELB"
    },
    "HostedZoneName": {
      "Type": "String",
      "Description": "Name of the hosted zone in Route 53"
    }
  },
  "Resources": {
    "WebELBRecordSet": {
      "Type": "AWS::Route53::RecordSet",
      "Properties": {
        "HostedZoneName": {
          "Fn::Join": [
            "",
            [
              {
                "Ref": "HostedZoneName"
              },
              "."
            ]
          ]
        },
        "Comment": "ELB alias resource record set",
        "AliasTarget": {
          "HostedZoneId": {
            "Fn::GetAtt": [
              "WebELB",
              "CanonicalHostedZoneNameID"
            ]
          },
          "DNSName": {
            "Fn::GetAtt": [
              "WebELB",
              "CanonicalHostedZoneName"
            ]
          }
        },
        "Type": "A",
        "Name": {
          "Fn::Join": [
            "",
            [
              {
                "Ref": "HostedZoneName"
              },
              "."
            ]
          ]
        }
      },
      "DependsOn": "WebELB"
    },
    "WebSG": {
      "Type": "AWS::EC2::SecurityGroup",
      "Properties": {
        "VpcId": {
          "Ref": "VPCId"
        },
        "GroupDescription": "Enable inbound access from the ELB",
      }
    },
    "WebSGIngress80": {
      "Type": "AWS::EC2::SecurityGroupIngress",
      "Properties": {
        "ToPort": "80",
        "IpProtocol": "tcp",
        "SourceSecurityGroupId": {
          "Ref": "ELBSG"
        },
        "GroupId": {
          "Ref": "WebSG"
        },
        "FromPort": "80"
      }
    },
    "WebSGIngress443": {
      "Type": "AWS::EC2::SecurityGroupIngress",
      "Properties": {
        "ToPort": "443",
        "IpProtocol": "tcp",
        "SourceSecurityGroupId": {
          "Ref": "ELBSG"
        },
        "GroupId": {
          "Ref": "WebSG"
        },
        "FromPort": "443"
      }
    },
    "ELBSG": {
      "Type": "AWS::EC2::SecurityGroup",
      "Properties": {
        "SecurityGroupIngress": [
          {
            "ToPort": "80",
            "IpProtocol": "tcp",
            "CidrIp": "0.0.0.0/0",
            "FromPort": "80"
          },
          {
            "ToPort": "443",
            "IpProtocol": "tcp",
            "CidrIp": "0.0.0.0/0",
            "FromPort": "443"
          }
        ],
        "VpcId": {
          "Ref": "VPCId"
        },
        "GroupDescription": "Enable inbound access to the ELB",
      }
    },
    "RDSSG": {
      "Type": "AWS::EC2::SecurityGroup",
      "Properties": {
        "VpcId": {
          "Ref": "VPCId"
        },
        "GroupDescription": "Enable inbound access only from the web server security group",
      }
    },
    "RDSSGIngress": {
      "Type": "AWS::EC2::SecurityGroupIngress",
      "Properties": {
        "ToPort": "3306",
        "IpProtocol": "tcp",
        "SourceSecurityGroupId": {
          "Ref": "WebSG"
        },
        "GroupId": {
          "Ref": "RDSSG"
        },
        "FromPort": "3306"
      }
    },
    "WebELB": {
      "Type": "AWS::ElasticLoadBalancing::LoadBalancer",
      "Properties": {
        "HealthCheck": {
          "HealthyThreshold": "2",
          "Interval": "30",
          "Target": "SSL:443",
          "Timeout": "5",
          "UnhealthyThreshold": "2"
        },
        "CrossZone": "true",
        "AvailabilityZones": {
          "Fn::GetAZs": ""
        },
        "SecurityGroups": [
          {
            "Ref": "ELBSG"
          }
        ],
        "Listeners": [
          {
            "InstancePort": "80",
            "LoadBalancerPort": "80",
            "Protocol": "HTTP"
          },
          {
            "InstancePort": "443",
            "Protocol": "HTTPS",
            "LoadBalancerPort": "443",
            "SSLCertificateId": {
              "Ref": "SSLCertificateId"
            },
            "InstanceProtocol": "HTTPS"
          }
        ]
      }
    },
    "WebELBWWWRecordSet": {
      "Type": "AWS::Route53::RecordSet",
      "Properties": {
        "HostedZoneName": {
          "Fn::Join": [
            "",
            [
              {
                "Ref": "HostedZoneName"
              },
              "."
            ]
          ]
        },
        "Comment": "ELB alias resource record set",
        "AliasTarget": {
          "HostedZoneId": {
            "Fn::GetAtt": [
              "WebELB",
              "CanonicalHostedZoneNameID"
            ]
          },
          "DNSName": {
            "Fn::GetAtt": [
              "WebELB",
              "CanonicalHostedZoneName"
            ]
          }
        },
        "Type": "A",
        "Name": {
          "Fn::Join": [
            "",
            [
              "www",
              ".",
              {
                "Ref": "HostedZoneName"
              },
              "."
            ]
          ]
        }
      },
      "DependsOn": "WebELB"
    }
  },
  "Outputs": {
    "ELBSGName": {
      "Description": "ELB Security Group name",
      "Value": {
        "Ref": "ELBSG"
      }
    },
    "WebSGName": {
      "Description": "Web Server Security Group name",
      "Value": {
        "Ref": "WebSG"
      }
    },
    "RDSSGName": {
      "Description": "RDS Security Group name",
      "Value": {
        "Ref": "RDSSG"
      }
    },
    "WebELBName": {
      "Description": "Web Server ELB name",
      "Value": {
        "Ref": "WebELB"
      }
    },
    "S3BucketName": {
      "Description": "S3 Bucket name",
      "Value": {
        "Ref": "ApplicationName"
      }
    }
  }
}
</pre>

rds

<pre>{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "MySQL RDS stack",
  "Parameters": {
    "RDSEngine": {
      "Type": "String",
      "Description": "The RDS engine type"
    },
    "RDSEngineVersion": {
      "Type": "String",
      "Description": "The RDS engine version"
    },
    "RDSDBName": {
      "AllowedPattern": "[a-zA-Z][a-zA-Z0-9]*",
      "Description": "The RDS name",
      "ConstraintDescription": "Must begin with a letter and contain only alphanumeric characters",
      "MinLength": "1",
      "MaxLength": "64",
      "Type": "String"
    },
    "RDSUsername": {
      "AllowedPattern": "[a-zA-Z][a-zA-Z0-9]*",
      "Description": "The RDS admin account username",
      "ConstraintDescription": "Must begin with a letter and contain only alphanumeric characters",
      "MinLength": "1",
      "MaxLength": "16",
      "Type": "String"
    },
    "RDSPassword": {
      "AllowedPattern": "[a-zA-Z0-9]*",
      "ConstraintDescription": "Must contain only alphanumeric characters",
      "NoEcho": "true",
      "Description": "The RDS admin account password",
      "MaxLength": "40",
      "MinLength": "1",
      "Type": "String"
    },
    "RDSDBPrefix": {
      "Type": "String",
      "Description": "The RDS table name prefix"
    },
    "RDSDBInstanceClass": {
      "Type": "String",
      "Description": "The RDS instance class"
    },
    "RDSAllocatedStorage": {
      "Type": "String",
      "Description": "The size of the storage allocated to the RDS"
    },
    "RDSAllowMajorVersionUpgrade": {
      "Type": "String",
      "Description": "Whether to allow major version upgrades to the RDS",
      "AllowedValues": [
        "true",
        "false"
      ]
    },
    "RDSAutoMinorVersionUpgrade": {
      "Type": "String",
      "Description": "Whether to allow automatic minor version upgrades to the RDS",
      "AllowedValues": [
        "true",
        "false"
      ]
    },
    "RDSPort": {
      "Type": "String",
      "Description": "The RDS port"
    },
    "VPCSecurityGroup": {
      "Type": "String",
      "Description": "The VPC security group used in the RDS instance"
    }
  },
  "Resources": {
    "RDSInstance": {
      "Type": "AWS::RDS::DBInstance",
      "Properties": {
        "DBParameterGroupName": {
          "Ref": "RDSDBParameterGroup"
        },
        "PubliclyAccessible": {
          "Ref": "RDSPubliclyAccessible"
        },
        "MasterUsername": {
          "Ref": "RDSUsername"
        },
        "VPCSecurityGroups": [
          {
            "Ref": "VPCSecurityGroup"
          }
        ],
        "Engine": {
          "Ref": "RDSEngine"
        },
        "AllowMajorVersionUpgrade": {
          "Ref": "RDSAllowMajorVersionUpgrade"
        },
        "AutoMinorVersionUpgrade": {
          "Ref": "RDSAutoMinorVersionUpgrade"
        },
        "PreferredBackupWindow": {
          "Ref": "RDSPreferredBackupWindow"
        },
        "BackupRetentionPeriod": {
          "Ref": "RDSBackupRetentionPeriod"
        },
        "DBName": {
          "Ref": "AWS::NoValue"
        },
        "PreferredMaintenanceWindow": {
          "Ref": "RDSPreferredMaintenanceWindow"
        },
        "EngineVersion": {
          "Ref": "RDSEngineVersion"
        },
        "DBSubnetGroupName": {
          "Ref": "RDSDBSubnetGroup"
        },
        "StorageType": {
          "Ref": "RDSStorageType"
        },
        "MasterUserPassword": {
          "Ref": "RDSPassword"
        },
        "DBInstanceClass": {
          "Ref": "RDSDBInstanceClass"
        },
        "Port": {
          "Ref": "RDSPort"
        },
        "DBInstanceIdentifier": {
          "Fn::Join": [
            "-",
            [
              {
                "Ref": "RDSDBName"
              }
            ]
          ]
        }
      },
      "DeletionPolicy": "Snapshot"
    }
  },
  "Outputs": {
    "RDSInstanceName": {
      "Description": "RDS database instance name",
      "Value": {
        "Ref": "RDSInstance"
      }
    },
    "RDSDBPrefix": {
      "Description": "RDS database prefix",
      "Value": {
        "Ref": "RDSDBPrefix"
      }
    },
    "RDSEndpointAddress": {
      "Description": "RDS database endpoint address",
      "Value": {
        "Fn::GetAtt": [
          "RDSInstance",
          "Endpoint.Address"
        ]
      }
    },
    "RDSEndpointPort": {
      "Description": "RDS database endpoint port",
      "Value": {
        "Fn::GetAtt": [
          "RDSInstance",
          "Endpoint.Port"
        ]
      }
    }
  }
}
</pre>

ec2

<pre>{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "EC2 stack",
  "Parameters": {
    "KeyName": {
      "Type": "String",
      "Description": "Name of the SSH Key used to access the instances"
    },
    "S3Bucket": {
      "Type": "String",
      "Description": "Name of the S3 Bucket"
    },
    "EC2InstanceType": {
      "ConstraintDescription": "Must use a valid instance type",
      "Type": "String",
      "Description": "Instance type to use for the EC2 instance"
    },
    "EC2InstanceName": {
      "Type": "String",
      "Description": "Instance name tag for the EC2 instance"
    },
    "AutoScalingDesiredCapacity": {
      "Type": "String",
      "Description": "Auto Scaling Group desired capacity"
    },
    "AutoScalingMinSize": {
      "Type": "String",
      "Description": "Auto Scaling Group minimum size"
    },
    "AutoScalingMaxSize": {
      "Type": "String",
      "Description": "Auto Scaling Group maximum size"
    },
    "SSLCertificateId": {
      "Type": "String",
      "Description": "The SSL certificate used on the ELB"
    },
    "InstanceAMI": {
      "ConstraintDescription": "Must be a valid AMI",
      "Type": "String",
      "Description": "AMI ID"
    },
    "HostedZoneName": {
      "Type": "String",
      "Description": "Name of the hosted zone in Route 53"
    },
    "RDSEndpointAddress": {
      "Type": "String",
      "Description": "The RDS endpoint"
    },
    "RDSEndpointPort": {
      "Type": "String",
      "Description": "The RDS endpoint port"
    },
    "RDSDBName": {
      "AllowedPattern": "[a-zA-Z][a-zA-Z0-9]*",
      "Description": "The RDS name",
      "ConstraintDescription": "Must begin with a letter and contain only alphanumeric characters",
      "MinLength": "1",
      "MaxLength": "64",
      "Type": "String"
    },
    "RDSUsername": {
      "AllowedPattern": "[a-zA-Z][a-zA-Z0-9]*",
      "Description": "The RDS admin account username",
      "ConstraintDescription": "Must begin with a letter and contain only alphanumeric characters",
      "MinLength": "1",
      "MaxLength": "16",
      "Type": "String"
    },
    "RDSPassword": {
      "AllowedPattern": "[a-zA-Z0-9]*",
      "ConstraintDescription": "Must contain only alphanumeric characters",
      "NoEcho": "true",
      "Description": "The RDS admin account password",
      "MaxLength": "40",
      "MinLength": "1",
      "Type": "String"
    },
    "RDSDBPrefix": {
      "Type": "String",
      "Description": "The RDS table name prefix"
    },
    "WebSG": {
      "Type": "String",
      "Description": "The web server security group"
    },
    "WebELB": {
      "Type": "String",
      "Description": "The web server ELB name"
    },
    "WebInstanceProfile": {
      "Type": "String",
      "Description": "The web server IAM Profile name"
    }
  },
  "Resources": {
    "WebLaunchConfiguration": {
      "Type": "AWS::AutoScaling::LaunchConfiguration",
      "Properties": {
        "UserData": {
          "Fn::Base64": {
            "Fn::Join": [
              "",
              [
                "#!/bin/bash\n",
                {
                  "Fn::Join": [
                    "",
                    [
                      "export AWS_DEFAULT_REGION=",
                      {
                        "Ref": "AWS::Region"
                      },
                      "\n"
                    ]
                  ]
                },
                {
                  "Fn::Join": [
                    "",
                    [
                      "/opt/aws/bin/cfn-init -s ",
                      {
                        "Ref": "AWS::StackId"
                      },
                      " -r WebLaunchConfiguration --region ",
                      {
                        "Ref": "AWS::Region"
                      },
                      "\n"
                    ]
                  ]
                },
                ". /etc/cloud-env.sh\n#yum install a bunch of stuff and use cloud-env.sh variables“
              ]
            ]
          }
        },
        "ImageId": {
          "Ref": "InstanceAMI"
        },
        "KeyName": {
          "Ref": "KeyName"
        },
        "SecurityGroups": [
          {
            "Ref": "WebSG"
          },
        ],
        "IamInstanceProfile": {
          "Ref": "WebInstanceProfile"
        },
        "InstanceType": {
          "Ref": "EC2InstanceType"
        }
      },
      "Metadata": {
        "AWS::CloudFormation::Init": {
          "config": {
            "files": {
              "/etc/cloud-env.sh": {
                "content": {
                  "Fn::Join": [
                    "",
                    [
                      "export AWSDefaultRegion=",
                      {
                        "Ref": "AWS::Region"
                      },
                      "\n",
                      "export S3Bucket=",
                      {
                        "Ref": "S3Bucket"
                      },
                      "\n",
                      "export HostedZoneName=",
                      {
                        "Ref": "HostedZoneName"
                      },
                      "\n",
                      "export DBName=",
                      {
                        "Ref": "RDSDBName"
                      },
                      "\n",
                      "export DBUsername=",
                      {
                        "Ref": "RDSUsername"
                      },
                      "\n",
                      "export DBPassword=",
                      {
                        "Ref": "RDSPassword"
                      },
                      "\n",
                      "export DBEndpoint=",
                      {
                        "Ref": "RDSEndpointAddress"
                      },
                      "\n",
                      "export DBPort=",
                      {
                        "Ref": "RDSEndpointPort"
                      },
                      "\n",
                      "export DBPrefix=",
                      {
                        "Ref": "RDSDBPrefix"
                      },
                      "\n",
                      "export INSTANCE_ID=$(/opt/aws/bin/ec2-metadata -i | cut -d ' ' -f 2)\n"
                    ]
                  ]
                },
                "owner": "root",
                "group": "root",
                "mode": "000400"
              }
            }
          }
        }
      }
    },
    "WebAutoScalingGroup": {
      "Type": "AWS::AutoScaling::AutoScalingGroup",
      "Properties": {
        "DesiredCapacity": {
          "Ref": "AutoScalingDesiredCapacity"
        },
        "Tags": [
          {
            "PropagateAtLaunch": "true",
            "Value": {
              "Fn::Join": [
                "-",
                [
                  {
                    "Ref": "EC2InstanceName"
                  }
                ]
              ]
            },
            "Key": "Name"
          }
        ],
        "LaunchConfigurationName": {
          "Ref": "WebLaunchConfiguration"
        },
        "MinSize": {
          "Ref": "AutoScalingMinSize"
        },
        "MaxSize": {
          "Ref": "AutoScalingMaxSize"
        },
        "LoadBalancerNames": [
          {
            "Ref": "WebELB"
          }
        ],
        "AvailabilityZones": {
          "Fn::GetAZs": ""
        }
      },
      "DependsOn": "WebLaunchConfiguration"
    }
  }
}
</pre>