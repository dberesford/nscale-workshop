Deploying on AWS
================

This tutorial covers:

1. Installing nscale on AWS
2. AWS configuration file updates
3. Deployment into AWS
4. Check and fix on AWS

During this tutorial we will assume deployment onto a linux based AMI. Specifically we assume use of ubuntu as the base operating system.

Installing nscale on AWS
------------------------
nscale should be installed on AWS in a similar manner to a direct linux install:

* boot up a community base VM. Do this using the ubuntu [EC2 AMI locator](http://cloud-images.ubuntu.com/locator/ec2/). We recommend using a 64 bit Trusty image.

* connect to your newly booted VM and install docker as per the instructions [here](http://docs.docker.com/installation/ubuntulinux/)

* add yourself to the docker group using: sudo usermod -G docker -a `whoami`

* Install node and npm from [here](http://nodejs.org/download/)

* Install git and configure:
    * git config --global user.name "your name"
    * git config --global user.email "you@somewhere.com"

Once the above dependencies have been met you can proceed to install nscale using 

    $sudo npm install -g nscasle

You should also make the confiugration changes as outlined [here](https://github.com/nearform/nscale). Specifically remember to set group permissions for the ubuntu user to allow access to docker commands without requiring sudo:

    sudo usermod -G docker -a `whoami`


Configuring AMIs for use with nscale
------------------------------------
A base AMI should be created that is configured for management by nscale. This is just a base image that has had docker installed, and the appropriate secrutiy permsission set. A great way of setting this up is to use the AWS configuration tools to create a private image of the system you have just installed. Do this now!

Once the server has been imaged, make a note of the ami identifier and log back into the running management system.

AWS configuration file updates
------------------------------
In order to operate correctly on AWS the nscale coniguration file requires some additional parameters. These are as follows:

* Kernel section
  * user - the username to use when connecting to remove systems (ubuntu)
  * identityFile - path to the ssh key file required to connect to remote systmes
  * region - the AWS region you are using
  * accessKeyId - your AWS API access key
  * secretAccessKey- your AWS secret access key
* Modules section
  * analysis section
    * set the analyzer to - nscale-aws-analyzer
  * Containers section
    * add aws-elb-container - specifiy the default subnet and vpc ID
    * add aws-sg-container - specifiy the default subnet and vpc ID
    * add aws-ami-container - specifiy the default subnet and vpc ID and also the ami id 

A full AWS confiuration file should look similar to the following:

```
{
  "kernel": {
    "systemsRoot": "/home/ubuntu/.nscale/data/systems",
    "user": "ubuntu",
    "identityFile": "PATH_TO_YOUR_KEYFILE.pem",
    "region": "YOUR_REGION",
    "accessKeyId": "YOUR_KEY",
    "secretAccessKey": "YOUR_SECRET_KEY"
  },
  "api": {
  },
  "web": {
  },
  "modules": {
    "protocol": {
      "require": "nscale-protocol",
      "specific": {
      }
    },
    "authorization": {
      "require": "nscale-noauth",
      "specific": {
        "credentialsPath": "/home/ubuntu/.nscale/data"
      }
    },
    "analysis": {
      "require": "nscale-aws-analyzer",
      "specific": {}
    }
  },
  "containers": [{
    "require": "aws-elb-container",
    "type": "aws-elb",
    "specific": {
      "region": "us-west-2",
      "defaultSubnetId": "subnet-nnn",
      "defaultVpcId": "vpc-nnn"
      }
  },
  {
    "require": "aws-sg-container",
    "type": "aws-sg",
      "specific": {
      "region": "us-west-2",
      "defaultSubnetId": "subnet-nnn",
      "defaultVpcId": "vpc-nnn"
    }
  },
  {
    "require": "aws-ami-container",
    "type": "aws-ami",
    "specific": {
      "region": "us-west-2",
      "defaultImageId": "ami-nnn",
      "defaultSubnetId": "subnet-nnn",
      "defaultVpcId": "vpc-nnn"
    }
  },
  {
    "require": "blank-container",
    "type": "blank-container",
    "specific": {}
  },
  {
    "require": "docker-container",
    "type": "docker",
    "specific": {
      "imageCachePath": "/tmp"
    }
  },
  {
    "require": "process-container",
    "type": "process",
    "specific": {}
  }]
}
```

Deploying into AWS
------------------
Now that we have configured our AWS setup lets proceed to deploy a system into AWS. To do this we will clone and deploy the startup death clock.

Lets get started by cloning the repository onto our AWS management system. Firstly cd into a working folder lets say /home/ubuntu/work/sudc.

	nsd system clone git@github.com:nearform/sudc-system.git

This will pull down the code for the Startup Death Clock system. In order to work with the AWS version of the system we will need to switch branches. to do this:

	cd /home/ubuntu/work/sudc/sudc-system
	git checkout aws

Lets take a look at the differences between the local configuration (on the master branch) and the AWS version:

####Infrastructure
The AWS version contains an additional file awsInfrastructure.js, which contains some AWS specific component definitions

 * awsWebElb - elastic load balancer definition
 * awsWebSg - security group definition
 * awsMachine - base AMI definition

####Topology
The AWS version also contains an updated topology which is specific for deployment into our AWS infrastructure. The topology section is as follows:

```
  aws: {
    awsWebElb: [{
      awsWebSg:[{awsMachine: ['web']},
                {awsMachine: ['doc', 'hist', 'real']}]
    }]
  },
```

The topology section defines how the service containers are distributed amongst machine instances.

####Edit
In order to make this work we need to make some minor adjustments to the ids specified in the configuration files:

* replace AMI-ID - open the file definitions/awsInfrastructure
  * under awsWebElb change the AvailabilityZone setting to match your availability zone
  * under awsWebSg change the VpcId setting to match your VPC
  * under awsMachine change the ImageId to match your ami id

####Compile

Lets now go ahead and compile for aws

	nsd system compile sudc aws

Now let's take a look at the system definition:

	nsd container list sudc

You should see the following containers:

	web                  docker
	hist                 docker
	real                 docker
	doc                  docker

####Build the system
Let's go ahead and build the containers ready for deployment:

	nsd container buildall sudc

Alternatively, you can build all the containers by themselves:

	nsd container build sudc hist
	nsd container build sudc real
	nsd container build sudc doc
	nsd container build sudc web

After those have all completed we should have four containers ready for deployment.

####Deploy the system
Now that we have an AWS system definintion and a set of containers to deploy, we can go ahead and push our system out onto AWS. to do this lets take a look at the local revision list

	nsd revision list

You should see something similar to the following:

```
revision             deployed who                                                     time                      description
136c840f016c57d0e23… false    John Doe <john.doe@gmail.com>                           2014-09-08T12:10:11.000Z  built container: 2b36df5faa5c92262aa675cd0a07312a…
31d0cff07829dc15e29… false    John Doe <john.doe@gmail.com>                           2014-09-08T12:09:29.000Z  built container: 51df875511be6f4951a1bd00610db2a9…
7e48d13a98746c8356a… false    John Doe <john.doe@gmail.com>                           2014-09-08T12:09:09.000Z  built container: f34344ef6f773c3e59b9cf84d01bf0ff…
1483ec749e9202dde10… false    John Doe <john.doe@gmail.com>                           2014-09-08T12:08:46.000Z  built container: 25a6d9868347b906345513aaf99e45ad…
5cab4567fef325bba9f… false    John Doe <john.doe@gmail.com>                           2014-09-08T12:06:24.000Z  first commit
3ebb8b5b986d76e59e9… false    Peter Elger <elger.peter@gmail.com>                     2014-09-08T08:16:00.000Z  added system definition
644891e6df77a8de7b2… false    Peter Elger <elger.peter@gmail.com>                     2014-09-07T18:56:23.000Z  first commit
```
Note that your specific output will differ to the above, particularly the revision numbers will be unique to your changes.

Lets first run a preview:

	nsd revision preview sudc <revision id>

Where revision id should be the id at the top of the revision list.

This will run an analysis against your configured AWS region and may take several seconds to complete. The preview should determine that a full deployment is required. nscale will show you a list of commands that will be executed against your infrastrcutre upon deployment. You should review this list to ensure that you are comfortable with the changes. If you are lets not go ahead and deploy.

	nsd revision deploy sudc <revision id>

nscale will no go ahead and deploy the SUDC system into your infrastructure. The following actions will be taken:

* create a new load balancer awsWebElb
* create a new security group awsWebSg
* spin up two machine instances
* deploy front end container to one of the machine instances and start it
* deploy 3 service containers to the other machine instnace and start them

Deployment may take several minutes depending on AWS machine instance spin up time. Once complete you should be able to point a web browser at port 8000 on the front end instance and view the startup death clock front end.

Check and Fix on AWS
--------------------
Now that we have a deployed working system on AWS, lets just check that everything is in order by running an nscale check:

	nsd system check sudc

The check command will run an analysis against the deployed system and compare it to your desired system configuration. This may take a few moments to run, but nscale should now respond that all is well with your deployment. Lets break something!

Go into the AWS management console and kill the SUDC front end machine. Now run the check again on the management host:

	nsd system check sudc

This time nscale should report that the system is broked and present a remedial action plan. To go ahead and fix the system execute:

	nsd system fix sudc

nscale will now boot up a replacement machine and deploy the appropriate containers to it, onece the fix has completed double check it by running:

	nsd system check sudc

Congratulations - you are now an nscale AWS ninja!