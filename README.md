# Sample project running GlusterFS in AWS

This setup uses Ansible to deploy and configure GlusterFS in AWS. The sample configuration will deploy 1 node in each AZ across us-west-2, this is done so we don't lose our cluster if an AZ goes out of service.  The nodes in the cluster are completely configurable, so it is easy to add/remove nodes and change the their zone placement.

**Here is a list of the components used in the deployment and how they come into play.**
* Ansible to provision resources from AWS and configure gluster once it is up.
* AWS managed ssh key to allow configuring gluster once our nodes have been provisioned
* Autoscaling group per node set to one instance. This will allow AWS to recover lost nodes for us.
* Launch configuration per node. This could easily be one launch configuration but breaking it out this way lets us change how each node is launched if we desired.
* EC2 with EBS running CoreOS.  CoreOS is very lightweight OS made for containers so using it to run Gluster in a Docker container fits really well.
  
> *Missing Features & Risks*
>  * Nodes need dns work so when a node goes down it can be added back to the pool as the same node
>  * Adding a node to the cluster will not add it to the volume
>  * EBS are not retained after termination so they cannot be attached to a new node
>  * Gluster configuration is done through the Docker API which is exposed over http, this is not a > recommended approach. This method is a risk and is used in this example until I work out using ssh from a remote machine into the container with Ansible.

---
## Getting Started

These instructions will get you a copy of the playbook up and running on your local machine allowing you to remote provision an AWS cluster.

### Prerequisites

* Python 2.7 or 3.5 and higher

> *The congfiguration for these is not in the project because they are usually setup once for all projects*
> * AWS role with an api key for provisioning.
> * AWS ssh key  
> * AWS VPC
> * AWS subnets (one per availability zone for the target region) 
> * Internet Gateway if configuring gluster from internet
>   * Subnet routing table needs to point to your Internet Gateway for public address CIDR.

### Installing

**Python dependencies used in AWS provisioning**
> Replace pip with pip3 if using python 3
* `pip install boto`
* `pip install botocore`
* `pip install boto3`

**Ansible for deployment and configuration**
* `pip install ansible`

### Configuration
This project only has one enviroment configured (`dev`)

**Adding nodes**  
*Add the unique node name (`ex. ga.4`) to the following groups in the hosts file*
* Desired subnet [ public-a | public-b | public-c ]
* provision

**Machine configuration**  
The relevant deployment configs are in the `group_vars/provision.yml` file
Property Key| Default | Description
------------|---------|------------
**gluster_version** | gluster4u0_centos7 | The Docker gluster/gluster-centos image tag
**ami_id** | ami-f4d4b694 (CoreOS-stable-1353.8.0-hvm) | Amazon machine image
**instance_type** | t2.micro | EC2 instance type
**security_groups** | [vpc, allow-public-ssh] | Array of the security group names the EC2 runs in. Deployment will look up actual security group id's that correspond to the configured named
**volumes** | device_name: /dev/xvda </br> volume_size: 30 </br> volume_type: gp2 </br> delete_on_termination: true| The volume type and size to attach to the EC2. If the device name is changed the cloud config template will need updated to reflect it
**cloud_config_template** | gluster_cloud_config | The config file CoreOS uses at boot time. Deployment will look for this file in templates project foler with an extension of `.j2`

**Gluster brick configuration**  
The `group_vars/config.yml` file holds brick configuration. The setup applies to the cluster, but the brick name could be defined per node  

Property Key | Default |Description
-------------|---------|-----------
**brick_dir** | /data/bricks | The volume that gluster will use to store it's bricks  
**arbiters** | 1 | The number of arbiters, helps avoid split-brain issues
**brick_type** | arbiter_brick_config |Chooses the template to create a indicated volume type
**brick_name** | sample | This brick name used with GlusterFS when creating a volume

---
## Deployment
The infrastructure is idempotent, so running the playbook multiple times will not reprovision anything. There are two classes of changes that will modify the deployment.

### Machine change, i.e. anything in the `provision.yml` file
* New launch config will be created
* Autoscaling Groups will swap out their launch configuration for the new one
* Each node will provision a new instance and once initialized will destroy the old one
* Old launch configuration will be deleted if unused
* GlusterFS configuration will run 
  * Create the brick directory if not already created
  * Connect any new peers
  * Create the volume if not already present
  * Mount the volume if not already mounted

### Config change or additional node, i.e. anything in the `config.yml` file
* Provision will run, but nothing will change since the state is correct
* GlusterFS configuration will run 
  * Create the brick directory if not already created
  * Connect any new peers
  * Create the volume if not already present
  * Mount the volume if not already mounted
 
### The playbook can be ran from the console with:  

``` shell
ansible-playbook -i inventories/dev gluster.yml --ask-vault-pass
```

> Playbook actions can be modified by appending some simple modifiers to the command above, here > are some examples:  
> * `--extra_vars "to_state=up` 
>   * Brings the gluster up, this is the default
> * `--extra_vars "to_state=down` 
>   * Tears down an existing cluster
> * `--limit public-a` 
>   * Restricts the deployment to a single AZ
