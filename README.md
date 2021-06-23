# SLURM Documentation
## Requirements
* All nodes should have a hostname that resolves to an easily identifiable name for the node, e.g. [compute01, compute02, etc..]
* All nodes should have password-less SSH configure between each all the other nodes in the cluster
* All nodes should be running Ubuntu 20.04
* All nodes should mount a NAS at the same location.

## Getting Started
Complete the following instructions on **controller** node of the cluster. This node will run the playbook and coordination the installation over all the other nodes in the cluster.
1. Install Ansible
2. Install Ansible Galaxy on the head node Using the following instructions: https://docs.ansible.com/ansible/latest/installation_guide/index.html
3. Create a file called `requirements.yml` file pointing to this git repository:

 ```sh
- name: gsl.slurm
  src: https://github.com/hadii-tech/gsl-slurm.git
  version: master

```
4. Install the following ansible galaxy roles by running the following command:
 
 ```sh
ansible-galaxy collection install ansible.posix
```

5. Create a file called `install.slurm` with the following content, update and modify node information as required in the `slurm_nodes` attribute. This attribute describes all the compute nodes in the cluster.

Also update the `slurm_nas` attribute based on where the NAS is mounted on each node in the cluster.
 
 ```sh
- name: Install slurm
   hosts: all
   roles:
     - gsl.slurm
   vars:
     slurm_nas: "/nas/slurm"
     slurm_nodes:
       - name: "slurmcompute01"
         CPUs: 1
         RealMemory: 1984
     slurm_partitions:
       - name: gsl
         Default: YES
         MaxTime: UNLIMITED
         Nodes: "slurmcompute01"


```
6. Lastly, update the inventory in `/etc/ansible/hosts` file should point to the appropriate nodes. Slurmservers contains all the nodes to be used as controller, while slurmexechosts represent all the nodes to be used as compute nodes. Finally, slurmdbdservers contains a list of all nodes to be used as database nodes. For a simple three node cluster, the following represents a sample `hosts` file that one may use.

 ```sh
[slurmservers]
slurmcontroller

[slurmexechosts]
Slurmcompute01

[slurmdbdservers]
slurmdb

```
 
## TASKS/common.yml:
This task installs the Slurm client on all three nodes, and configures a number of common settings all the nodes will need:
* Setup Log rotation 
  * Rotatest slurm logs to avoid filling up disk space
* Install slurm.conf 
  *	Installs slurm.conf which describes general Slurm configuration information, the nodes to be managed, information about how those nodes are grouped into partitions, and various scheduling parameters associated with those partitions. This task is installed by root owner, and once completed notifies “restart slurmd” and “restart slurmctld” ## Tasks.
* Setup munge
   * Munge is used for authentication communication between nodes. In this step, the munge key from the controller node is copied to all other nodes in the cluster.

## TASKS/db.yml:
* This task sets up and configures a mariadb database to be used for SLURM accounting.

## TASKS/docker.yml:
 * Ensure repository key is installed
     *	This task downloads the key from the docker url to start the docker installation process.
 * Ensure docker registry is available 
     *This task uses the “apt_repository” module which installs the necessary packages from a docker repository to carry out the docker installation.
* Install docker
  * This task triggers the docker installation.


## TASKS/main.yml:
* Include user Tasks
  * This task calls on the “user.yml” file for the purposes of creating a `slurm` linux user and group across all nodes.
* Include controller installation
   * This task calls on the “slurmctld.yml” file for the configuration of the controller node. 
* Incude execution host installation # Tasks
   * Calls on the “slurmd.yml” file for the configuration of the execution host node.
* Include DB installation # Tasks
   * Calls on the “slurmdbd.yml” file for the configuration of the database server.
*Import common # Tasks
   * Calls on the “common.yml” file to run the tasks that are common to all nodes in the cluster
* Ensure slurmdbd is enabled and running
   * Uses the ansible service module to ensure that the slurmdbd node is enabled and running. Slurmdbd provides a secure enterprise-wide interface to a database for Slurm. This is particularly useful for archiving accounting records.
* Ensure slurmctld is enabled and running
   * Uses the ansible service module to ensure that the slurmctld node is enabled and running. slurmctld is the central management daemon of Slurm. It monitors all other Slurm daemons and resources, accepts work (jobs), and allocates resources to those jobs.
* Ensure slurmd is enabled and running
   *Uses the ansible service module to ensure that the exec node is enabled and running. Slurmd or exec node is the compute node daemon of Slurm. 

## TASKS/munge.yml:
*Check munge dir
  * This task ensure the munge key directory “/etc/munge” exists.
* Get munge file stat
   * Determines if a munge key file exists on the local host that runs the playbook, in this case, this is our main SLURM head node.
* Copy munge key to targets
   * Copies the munge key from the local host to all the other nodes in the cluster.
*Ensure Munge is enabled and running
  This task uses the ansible service module to ensure munge is enabled and running.
  
## TASKS/slurmctld.yml:
* Install Slurm controller packages
    * Installs the necessary slurm packages the controller node requires to operate.
*Create slurm state directory
    * Creates the state directory, which is used when the slurm controller node is shutting down, as it saves it’s current state to the state directory. The task also has a “when” condition, which causes the controller node to reload once the state directory has been created.
*  Create slurm log directory
    *Creates the log file location for the controller node, which is the slurmctld log directory. 
* 'Service slurmctld : Directory of Pidfile must exist and set slurm group permission'
   * Ensures the directory for the slurmctld service exists and sets group permission on the directory appropriately. 

## TASKS/slurmd.yml:
* Install Slurm execution host packages
   * Installs the necessary slurm packages for the “exec” node.  
* Create slurm spool directory
    * Creates the slurm spool directory which holds all work requests and all log. When the directory is created, a handler is triggered and “slurmd” is reloaded. 
* Create slurm log directory
    *	Creates the core file location for the exec node, which is the slurm log directory. 
* Create slurm pid directory
    * Creates the slurm pid directory in which the pid files for the configuration will be downloaded. 
* Include config dir creation # Tasks
 
 
## TASKS/configure_cluster.yml:
* Check if main slurm cluster called `cluster` already exists in the accounting database.
* If it doesn't already exist, it is created
* Commercial job Quality of Service (qos) is created with a job priority of 100.

