# Getting started with Aliyun CLI

## Installation
Check out the guideline here: https://github.com/aliyun/aliyun-cli ( Folks can't read Chinese may need to use Google translator)

Install python pip if necessary
```
curl https://bootstrap.pypa.io/get-pip.py | sudo python
```

Install aliyuncli
```
sudo pip install aliyuncli
```

Install some sdk packages:
```
sudo pip install "aliyun-python-sdk-ecs"
sudo pip install "aliyun-python-sdk-rds"
sudo pip install "aliyun-python-sdk-oss"
sudo pip install "aliyun-python-sdk-slb"
```

### Confirm all SDKs available via aliyuncli
```
aliyuncli
```

The ouput will look like below
```
ecs                                       | oss
rds                                       | slb
```

### Configure access Key ID and access key secret (Find them in your console menu):

```
aliyuncli configure
# Aliyun Access Key ID [None]: <Your access key ID>
# Aliyun Access Key Secret [None]: <Your access key secret>
# Default Region Id [None]: cn-hangzhou
# Default output format [None]: json
```
**NOTE**: Without specifying the default region ID, you won't be able to run any commands because the CLI doesn't know what API endpoint it should connect to.

## Use ECS API to create a new instance

New let's check all available ECS commands and see some help
```
aliyuncli ecs
```

### Issue 1 - Required parameters

Find out how to use `CreateInstance` command by typing
```
aliyuncli ecs CreateInstance help
```

The help content is not so helpful as **it doesn't tell which are the required parameters**...
Luckily we can check the [API documentation](https://intl.aliyun.com/docs#/pub/ecs_en_us/open-api/instance&createinstance) to see that following parameters are required to create a new instance:
- RegionId
- ImageId
- InstanceType
- SecurityGroupId

Since we have no idea what are availabe for use, check available zones, images, instance types and security groups like below (NOTE: All **LocalName**s or **Description**s are not understandable because they are in Chinese)
```
aliyuncli ecs DescribeRegions
aliyuncli ecs DescribeImages
aliyuncli ecs DescribeInstanceTypes
```

### Issue 2 - Available images at regions are incorrect?

By having a quick look at what kind of instances are available in each regions. There are many differences between the regions' image lists:

```
$ aliyuncli ecs DescribeImages --RegionId cn-hangzhou | grep -e "ImageId"
    "ImageId": "ubuntu1404_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "ubuntu1404_32_40G_aliaegis_20160222.vhd", 
    "ImageId": "ubuntu1204_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "ubuntu1204_32_40G_aliaegis_20160222.vhd", 
    "ImageId": "suse11sp3_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "suse11sp2_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "suse11sp1_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "gentoo13_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "debian750_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "debian750_32_40G_aliaegis_20160222.vhd", 
$ aliyuncli ecs DescribeImages --RegionId ap-southeast-1 | grep -e "ImageId"
    "ImageId": "centos7u0_64_40G_aliaegis_20160120.vhd", 
    "ImageId": "win2012_64_dataCtr_R2_en_40G_alibase_20151212.vhd", 
    "ImageId": "win2012_64_dataCtr_R2_cn_40G_alibase_20151212.vhd", 
    "ImageId": "win2008_64_ent_r2_en_40G_alibase_20151212.vhd", 
    "ImageId": "win2008_64_ent_r2_cn_40G_alibase_20151212.vhd", 
    "ImageId": "win2008_32_stand_sp2_cn_40G_alibase_20151212.vhd", 
    "ImageId": "opensuse1301_64_40G_aliaegis_20160120.vhd", 
    "ImageId": "opensuse1301_32_40G_aliaegis_20160120.vhd", 
    "ImageId": "freebsd1001_64_40G_aliaegis_20160120.vhd", 
    "ImageId": "centos5u8_64_40G_aliaegis_20160120.vhd", 
$ aliyuncli ecs DescribeImages --RegionId cn-shenzhen | grep -e "ImageId"
    "ImageId": "ubuntu1404_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "ubuntu1404_32_40G_aliaegis_20160222.vhd", 
    "ImageId": "ubuntu1204_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "ubuntu1204_32_40G_aliaegis_20160222.vhd", 
    "ImageId": "suse11sp3_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "suse11sp2_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "suse11sp1_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "gentoo13_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "debian750_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "debian750_32_40G_aliaegis_20160222.vhd", 
$ aliyuncli ecs DescribeImages --RegionId cn-beijing | grep -e "ImageId"
    "ImageId": "ubuntu1404_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "ubuntu1404_32_40G_aliaegis_20160222.vhd", 
    "ImageId": "ubuntu1204_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "ubuntu1204_32_40G_aliaegis_20160222.vhd", 
    "ImageId": "suse11sp3_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "suse11sp2_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "suse11sp1_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "gentoo13_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "debian750_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "debian750_32_40G_aliaegis_20160222.vhd", 
$ aliyuncli ecs DescribeImages --RegionId cn-shanghai | grep -e "ImageId"
    "ImageId": "ubuntu1404_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "ubuntu1404_32_40G_aliaegis_20160222.vhd", 
    "ImageId": "ubuntu1204_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "ubuntu1204_32_40G_aliaegis_20160222.vhd", 
    "ImageId": "suse11sp3_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "suse11sp2_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "suse11sp1_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "gentoo13_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "debian750_64_40G_aliaegis_20160222.vhd", 
    "ImageId": "debian750_32_40G_aliaegis_20160222.vhd", 
$ aliyuncli ecs DescribeImages --RegionId cn-hongkong | grep -e "ImageId"
    "ImageId": "centos7u0_64_40G_aliaegis_20160120.vhd", 
    "ImageId": "win2012_64_dataCtr_R2_en_40G_alibase_20151212.vhd", 
    "ImageId": "win2012_64_dataCtr_R2_cn_40G_alibase_20151212.vhd", 
    "ImageId": "win2008_64_ent_r2_en_40G_alibase_20151212.vhd", 
    "ImageId": "win2008_64_ent_r2_cn_40G_alibase_20151212.vhd", 
    "ImageId": "win2008_32_stand_sp2_cn_40G_alibase_20151212.vhd", 
    "ImageId": "opensuse1301_64_40G_aliaegis_20160120.vhd", 
    "ImageId": "opensuse1301_32_40G_aliaegis_20160120.vhd", 
    "ImageId": "freebsd1001_64_40G_aliaegis_20160120.vhd", 
    "ImageId": "centos5u8_64_40G_aliaegis_20160120.vhd", 
$ aliyuncli ecs DescribeImages --RegionId us-west-1 | grep -e "ImageId"
    "ImageId": "centos7u0_64_40G_aliaegis_20160120.vhd", 
    "ImageId": "win2012_64_dataCtr_R2_en_40G_alibase_20151212.vhd", 
    "ImageId": "win2012_64_dataCtr_R2_cn_40G_alibase_20151212.vhd", 
    "ImageId": "win2008_64_ent_r2_en_40G_alibase_20151212.vhd", 
    "ImageId": "win2008_64_ent_r2_cn_40G_alibase_20151212.vhd", 
    "ImageId": "win2008_32_stand_sp2_cn_40G_alibase_20151212.vhd", 
    "ImageId": "opensuse1301_64_40G_aliaegis_20160120.vhd", 
    "ImageId": "opensuse1301_32_40G_aliaegis_20160120.vhd", 
    "ImageId": "freebsd1001_64_40G_aliaegis_20160120.vhd", 
    "ImageId": "centos5u8_64_40G_aliaegis_20160120.vhd",
```

I am curious whether the results above means that `I can't create an ECS instance running CentOS in Shanghai`? The answer is **NO**. From the web console of `int.aliyun.com`, I was able to create it! So it seems like `aliyuncli` doesn't give us the correct information.


### Try creating a new security group
Create an empty security group in order to create the new instance. We can get back to configure it later:
```
aliyuncli ecs CreateSecurityGroup

# The response will look like below
{
    "SecurityGroupId": "sg-23ewnvyt4",
    "RequestId": "A113C5FA-4C46-46C9-A02B-70CCCFD6AD9F"
}
```

Configure policies for the security group:

```
aliyuncli ecs AuthorizeSecurityGroup --IpProtocol all --SecurityGroupId sg-23ewnvyt4 --PortRange "-1/-1" --SourceCidrIp "0.0.0.0/0" --Policy accept --NicType internet

# TODO: Add a command to configure internet outflow connection
```

**ISSUE 1**: Can't configure internet outflow connection

**ISSUE 2** It's quite funny when an error occurs. For example the error message below doesn't relate to the error code at all! The message looks ok, but the error code is completely wrong!

```
aliyuncli ecs AuthorizeSecurityGroup --IpProtocol tcp --SecurityGroupId sg-28heoi062 --PortRange '-1/-1' --SourceCidrIp "0.0.0.0/0" --Policy accept --NicType internet
{
    "Code": "InvalidIpProtocol.Malformed", 
    "Message": "The specified parameter \"PortRange\" is not valid.", 
    "HostId": "ecs-cn-hangzhou.aliyuncs.com", 
    "RequestId": "81D2C6A5-BC4B-4040-BFAE-A39567286A85"
}
```

### Create a ECS instance and modify some attributes

Now we are ready to create a new ECS instance!!

```
aliyuncli ecs CreateInstance \
  --ImageId centos7u0_64_20G_aliaegis_20150130.vhd \
  --InstanceType ecs.t1.small \
  --InstanceName generated-by-cli \
  --Description "This is pretty cool" \
  --SecurityGroupId sg-23ewnvyt4
  
# The output will be the created instance's information:
{
    "InstanceId": "i-23yi8tnhj",
    "RequestId": "36D8C4EC-2E38-4468-B4F4-AE658B90E1C3"
}
```

Now confirm the instance's information:
```
aliyuncli ecs DescribeInstanceAttribute --InstanceId i-23yi8tnhj

# OR: aliyuncli ecs DescribeInstanceStatus
```

The instance status is now "Stopped". Starting it is quite straight forward
```
aliyuncli ecs StartInstance --InstanceId i-23yi8tnhj
```

Curious about what can be changed later? Try this
```
aliyuncli ecs | grep Modify
```

Modify `root` password:
```
aliyuncli ecs ModifyInstanceAttribute --Password "<Your root password>" --InstanceId "i-23yi8tnhj"
aliyuncli ecs RebootInstance --InstanceId "i-23yi8tnhj"
```

### Network configuration

At this point **the new ECS instance can't be accessed using `ping` nor `SSH`**. Right, we haven't assigned a public IP to it!

```
# Give the instance a public IP
aliyuncli ecs AllocatePublicIpAddress --InstanceId i-23yi8tnhj

# The IP address will be returned in the response:
{
    "IpAddress": "xx.yy.zz.tt", 
    "RequestId": "CC9A1897-64AA-46BA-8B99-1A59EC255562"
}
```

Looks ok, but... nothing changes? Why? According to the ECS API documentation, we need to specify the chargeType and max bandwidth for the instance. Let's give it a try:

```
aliyuncli ecs ModifyInstanceNetworkSpec --InternetMaxBandwidthOut 100 --InstanceId i-283shc9ic
aliyuncli ecs ModifyInstanceAttribute --InternetChargeType PayByTraffic --InstanceId i-283shc9ic
```

Still no luck ...

TO BE CONTINUED...
