Ansible 2.7

Requirements
AWS modules need python boto module `pip install boto, botocore, boto3`

# Sample project running GlusterFS in AWS

This setup uses Ansible to deploy and configure a gluster in AWS. The sample configuration here will deploy 1 node in each AZ across us-west-2, this is done so we don't lose our cluster if an AZ goes out of service.  The nodes in the cluster are completely configurable, so it is easy to add/remove nodes and change the their zone placement. This sample is running Ansible locally and connecting to AWS over ssh. Ansible should really be run in AWS or at a minimum through a VPN.

`The gluster confgiuration still needs some work, Asible ssh connection to the node needs to proxy to docker exec -ti $CONTAINER sh`

**Here is a list of the components used in the deployment and how they come into play.**
* Ansible to provision resources from AWS and configure gluster once it is up.
* AWS managed ssh key.  This is used for configuring gluster once our nodes have been provisioned
* Route53 in a private hosted zone.  These records are the server names gluster will use when doing a probe. Because we aren't using a load balancer the route points directly to the instance.  This should change to a lambda that can subscribe to sns with autoscale messages to update the route.
* Autoscaling group per server set to one instance.  This will allow AWS to recove lost nodes for us.  The reason for 1 is to allow the lambda to use the tags to determine the Route53 record to update, if there was more than one it would break the mapping of route to ec2 inside of gluster
* Launch configuration per server.  This could easily be one launch configuration but breaking it out this way lets us change how each server is launched if we desired.
* EC2 with EBS running CoreOS.  CoreOS is very lightweight OS made for containers so using it to run Gluster in a Docker fits really well.

## Getting Started

These instructions will get you a copy of the project up and running on your local machine for development and testing purposes. See deployment for notes on how to deploy the project on a live system.

### Prerequisites

* Python 2.7 or 3.5 and higher
* Pyhton packages boto, botocore and boto3.  These are used for interacting with AWS.
* [Ansible 2.7 or higher](https://docs.ansible.com/ansible/2.7/installation_guide/intro_installation.html)

**The congfiguration for these is not in the project because they are usually setup once for all projects**
* AWS role with an api key for provisioning.
* AWS ssh key
* Private hosted Host Zone.  
* AWS VPC
* AWS subnets (one per availability zone for the target region). 
* Internet Gateway if configuring gluster from internet
  * Subnet routing table needs to point to your Internet Gateway for public address CIDR.

```
Give examples
```

### Installing

A step by step series of examples that tell you how to get a development env running

Say what the step will be

```
Give the example
```

And repeat

```
until finished
```

End with an example of getting some data out of the system or using it for a little demo

## Running the tests

Explain how to run the automated tests for this system

### Break down into end to end tests

Explain what these tests test and why

```
Give an example
```

### And coding style tests

Explain what these tests test and why

```
Give an example
```

## Deployment

Add additional notes about how to deploy this on a live system

## Built With

* [Dropwizard](http://www.dropwizard.io/1.0.2/docs/) - The web framework used
* [Maven](https://maven.apache.org/) - Dependency Management
* [ROME](https://rometools.github.io/rome/) - Used to generate RSS Feeds
