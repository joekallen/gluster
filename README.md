# Sample project running GlusterFS in AWS

This setup uses Ansible to deploy and configure gluster in AWS. The sample configuration will deploy 1 node in each AZ across us-west-2, this is done so we don't lose our cluster if an AZ goes out of service.  The nodes in the cluster are completely configurable, so it is easy to add/remove nodes and change the their zone placement.

## Missing features
* Nodes need dns work so when a node goes down it can be added back to the pool as the same node
* EBS are not retained after termination so they cannot be attached to a new node

**Here is a list of the components used in the deployment and how they come into play.**
* Ansible to provision resources from AWS and configure gluster once it is up.
* AWS managed ssh key to allow configuring gluster once our nodes have been provisioned
* Autoscaling group per server set to one instance. This will allow AWS to recover lost nodes for us.
* Launch configuration per server. This could easily be one launch configuration but breaking it out this way lets us change how each server is launched if we desired.
* EC2 with EBS running CoreOS.  CoreOS is very lightweight OS made for containers so using it to run Gluster in a Docker container fits really well.

## Getting Started

These instructions will get you a copy of the playbook up and running on your local machine allowing you to remote provision an AWS cluster.

### Prerequisites

* Python 2.7 or 3.5 and higher

---

*The congfiguration for these is not in the project because they are usually setup once for all projects*
* AWS role with an api key for provisioning.
* AWS ssh key  
* AWS VPC
* AWS subnets (one per availability zone for the target region). 
* Internet Gateway if configuring gluster from internet
  * Subnet routing table needs to point to your Internet Gateway for public address CIDR.

### Installing
*Replace pip with pip3 if using python 3*

**Python dependencies used in AWS provisioning**
* pip install boto
* pip install botocore
* pip install boto3

**Ansible for deployment and configuration**
* pip install ansible

### Configuration
This project only has one enviroment configured

**Adding nodes**  
*Add the unique server name (ex. ga.4) to the following groups in the hosts file*
* Desired subnet ( public-a, public-b or public-c)
* provision

**Machine configuration**  
The relevant deployment configs are in the group_vars/provision.yml file.
* **gluster_version** - The tag is use for the gluster/gluster-centos docker image
* **ami_id** - Amazon machine image
  * default is latest version of CoreOS
* **instance_type** - EC2 instance type
* **security_groups** - Array of the security group names the EC2 runs in
  * The dpeloyment will look up the ids for the security groups, this allows the config to remain readable
* **volumes** - The volume type and size to attach to the EC2
  * If the device name is changed the cloud config template will need updated to reflect it
* **cloud_config_template** - The config file CoreOS uses at boot time

**Gluster brick configuration**  
The group_vars/config.yml file holds brick configuration. The setup applies to the cluster, but the brick name could be defined per server and more configs could be added in host_vars and then just delegated to the server it applies to
* **brick_dir** - The volume that gluster will use to store it's bricks  
* **arbiters** - The number of arbiters, helps avoid split-brain issues
* **brick_type** - Chooses the template to create a indicated volume type
* **brick_name** - This is the brick name
*  
## Deployment

The playbook can be ran from the console with:  
`ansible-playbook -i inventories/dev gluster.yml --ask-vault-pass`

Playbook actions can be modified by appending some simple modifiers to the command above, here are some examples:  
* `--extra_vars "to_state=up` 
  * Brings the gluster up, this is the default
* `--extra_vars "to_state=down` 
  * Tears down an existing cluster
* `--limit public-a` 
  * Restricts the deployment to a single AZ
