
# Introduction

Creating a separate EBS volume can be very useful to prevent application data loss in the event your EC2 instance is unexpectedly terminated. It also makes it very easy to back up your application data by snapshotting the EBS volume.

AWS makes it easy in CloudFormation templates to add additional EBS volumes to an EC2 Instance,
but it's not obvious from the AWS CloudFormation docs how to map an EBS volume
to a Linux mount point, like `/var/myapp`.

In this article I'll show you how to add an EBS volume to an EC2 instance and
automatically mount the EBS volume to a directory/mount point inside the instance.

# Deploying the CloudFormation template

The example CloudFormation template we'll be using is at
[1Strategy/blog-add-ebs-volume](https://github.com/1Strategy/blog-add-ebs-volume).

If you'd like to try it out, clone the git repo and run this AWS CLI command to deploy it:

```
aws cloudformation deploy --template-file ./lvm-volume.yaml --stack-name lvm-volume --parameter-overrides VpcIdParameter=vpc-abcd1234 InstanceSubnetIdParameter=subnet-abcd1234 SshKeyParameter=mysshkey
```

# How to add an EBS volume in CloudFormation

EBS volumes are block storage devices, and adding a block device to an instance requires
only a [few lines of code in CloudFormation](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-blockdev-mapping.html).


```YAML
  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      BlockDeviceMappings:
        # Create a separate volume
        - DeviceName: /dev/sdf
          Ebs:
            DeleteOnTermination: false
            VolumeSize: 10
```

#### AWS Block Device naming conventions

In the example above, I've named the block device `/dev/sdf`. See [Device Naming on Linux instances](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/device_naming.html) for more details on
naming conventions for Linux block devices. In short, you should use `sdf - sdp`.

Note that the OS, e.g. Amazon Linux, will rename `/dev/sdf` to `/dev/xvdf` (Xen Virtual Device "F"). For an explanation, see [Device Name Considerations](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/device_naming.html#device-name-limits) in the AWS EC2 User Guide.

#### EBS Volume Attached But Not Usable

If you deploy your CloudFormation template with this block device mapping, the EBS
volume will be attached to your EC2 instance, but you won't be able to use it until
you create a file system on it and mount it to a directory, or mount point, e.g.
`/var/myapp`.


# How to mount a volume in Linux automatically

You can of course SSH in to your instance after creation and run the commands to
create and mount a file system by hand, but if you are already using CloudFormation
you want it all automated. In my case, I wanted to mount the EBS volume under
`/var/myapp` so my application could store its data on a separate volume which
wouldn't disappear if the instance was terminated.


#### Running File System Create and Mount commands

The CloudFormation snippet below uses AWS [CloudFormation::Init](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-init.html), combined with
[`cfn-init`](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-init.html) in the `UserData`
section to run the commands to create and mount the file system at boot.

Note that the commands are run in alphabetical order by name, *not* in the order listed in
the `commands` section. That is why I've prepended each command with a number,
e.g. `1_pvcreate`, `2_vgcreate`, etc.

#### LVM
I'm using [LVM](https://wiki.archlinux.org/index.php/LVM), or Logical Volume Manager,
to make it easier to resize logical disk volumes in the future if need be.

The `AWS::CloudFormation::Init` commands below first create an LVM physical volume, then a volume group, then a
logical volume.

#### File System
After that, we create an [ext4](https://en.wikipedia.org/wiki/Ext4)
file system, make the `/var/myapp` directory, and add the mount point to the
`/etc/fstab` file so it will be mounted every time the system boots.

Finally, we run the `mount -a` command to mount the newly added logical volume
to the `/var/myapp` directory.

```YAML
  EC2Instance:
    Type: AWS::EC2::Instance
    Metadata:
      AWS::CloudFormation::Init:
        config:
          commands:
            1_pvcreate:
              command: pvcreate /dev/xvdf
            2_vgcreate:
              command: vgcreate vg0 /dev/xvdf
            3_lvcreate:
              command: lvcreate -l 100%FREE -n myapp vg0
            4_mkfs:
              command: mkfs.ext4 /dev/vg0/myapp
            5_mkdir:
              command: mkdir /var/myapp
            6_fstab:
              command: echo "/dev/mapper/vg0-myapp /var/myapp ext4 defaults 0 2" >> /etc/fstab
            7_mount:
              command: mount -a
    Properties:
      BlockDeviceMappings:
        <snip>
      UserData:
        Fn::Base64: !Sub |
         #!/usr/bin/env bash
         set -o errexit
         yum -y update aws-cfn-bootstrap
         /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource EC2Instance --region ${AWS::Region}
         /opt/aws/bin/cfn-signal --exit-code $? --stack ${AWS::StackName} --resource EC2Instance --region ${AWS::Region}

```

#### UserData -> cfn-init -> AWS::CloudFormation::Init

In the snippet above, the `UserData` section is run once at OS first boot as a Bash script.
It calls `cfn-init` which triggers the `AWS::CloudFormation::Init` section in the
Metadata section above. This is the recommended pattern for running bootstrap commands on
your Linux instance. See [AWS::CloudFormation::Init](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-init.html)
for more details.

#### Note
If you are installing an application package which will create directories under
your `/var/myapp` mount point, be sure to run the commands to create the mount
point before installing your package. Otherwise the sub-directories the package
creates will be wiped out.

# Results

After running the complete CloudFormation template you will have the `/var/myapp`
directory mounted on your `/dev/sdf` (or `/dev/xvdf`) EBS volume. If your instance
disappears, your data will not be lost. You can spin up another EC2 instance and
mount the orphaned volume to recover your data. Note that once you've created
a file system and LVM volumes you won't need to create them again for that EBS
volume.

You can view information about your EBS volume and mount point by running the
`lsblk` command on your instance. Run `man lsblk` for more info.

```
$ lsblk --fs /dev/sdf

NAME        FSTYPE      LABEL UUID             MOUNTPOINT
xvdf        LVM2_member       jgzpM5-...
└─vg0-myapp ext4              165452b3-...     /var/myapp
```

# Summary

The full example CloudFormation template is at
[1Strategy/blog-add-ebs-volume](https://github.com/1Strategy/blog-add-ebs-volume).

Creating a separate EBS volume can be very useful to prevent application data loss
in the event your EC2 instance is unexpectedly terminated. It also makes it very
easy to back up your application data by snapshotting the EBS volume. Finally,
it cleanly separates your application data from your OS root volume.

In this post, I showed you how to create an EC2 Instance via a AWS CloudFormation
template with an EBS volume to store your application data separate from your EC2
instance's root volume.

You also learned how to create an LVM volume and ext4 filesystem on your EBS volume,
and how to auto-mount the file system at boot by adding a line to your `/etc/fstab`
file.

I hope this helps you improve your CloudFormation templates when dealing with EBS volumes for applications.
