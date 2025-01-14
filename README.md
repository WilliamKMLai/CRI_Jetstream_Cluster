# Elastic Slurm Cluster on the Jetstream Cloud

## Intro

This repo contains scripts and ansible playbooks for creating a virtual 
cluster in an Openstack environment, specifically aimed at the XSEDE 
Jetstream resource.

The basic structure is to have a single image act as headnode, with
compute nodes managed by SLURM via the openstack API. A customized 
image is created for worker nodes, which contains configuration 
and software specific to that cluster. The Slurm daemon on the
headnode dynamically creates and destroys worker nodes in response to 
jobs in the queue. The current version is based on CentOS 8, using
RPMs from the [OpenHPC project](https://openhpc.community).

## Current Useage
To build your own Virtual cluster, starting on your localhost:

1. If you don't already have an openrc file, see the 
   [Jetstream Wiki](https://wiki.jetstream-cloud.org).

1. Clone this repo.

1. Know the path to an openrc file for the allocation in which you'd like to create a 
   virtual cluster to this repo. 

1. If you'd like to modify your cluster, now is a good time!
   This local copy of the repo will be re-created on the headnode, but
   if you're going to use this to create multiple different VCs, it may be 
   preferable to make the following modifications in seperate files.
   * The number of nodes can be set in the slurm.conf file, by editing
   the NodeName and PartitionName line. 
   * If you'd like to change the default node size, the ```node_size=```line 
     in ```slurm_resume.sh``` must be changed.
     This should take values corresponding to instance sizes in Jetstream, like
     "m1.small" or "m1.large". Be sure to edit the ```slurm.conf``` file to 
     reflect the number of CPUs available.
   * If you'd like to enable any specific software, you should edit 
     ```compute_build_base_img.yml```. The task named "install basic packages"
     can be easily extended to install anything available from a yum 
     repository. If you need to *add* a repo, you can copy the task
     titled "Add OpenHPC 1.3.? Repo". For more detailed configuration,
     it may be easiest to build your software in /export on the headnode,
     and only install the necessary libraries via the compute_build_base_img
     (or ensure that they're available in the shared filesystem).
   * For other modifications, feel free to get in touch!

1. Run ```cluster_create.sh``` - it *will* require an ssh key to exist in
   ```${HOME}/.ssh/id_rsa.pub```. This will be the key used for your jetstream
   instance! If you prefer to use a different key, be sure to edit this
   script accordingly. The expected argument is only the headnode name, 
   and will create an 'm1.small' instance for you. The path the your openrc 
   defaults to the same directory as the script. *WARNING: Openstack may replace 
   underscores in the <headnode-name> with dashes, causing various failures later on.*
   

   ```./cluster_create.sh -n <headnode-name> -o <openrc-path>```

   Watch for the ip address of your new instance at the end of the script!
   It is worth double-checking the output of the script to see that the ansible
   playbook for compute image creation ran successfully - no failed tasks means
   that the image for your compute nodes should be available.
   If you would like to create a cluster with a shared volume (mounted 
   on /export), you may use:

   ```./cluster_create.sh -n <headnode-name> -v <volume-size-in-GB>```
   
   You may also create a larger headnode via the -s option, which takes sizes
   based on the flavours used in your Openstack cloud - for example:

   ```./cluster_create.sh -n <headnode-name> -s m2.xlarge```

1. The headnode_create script has copied everything in this directory 
   to your headnode EXCEPT your local openrc file. You should now be able to ssh in
   as the centos user, with your default ssh key: 
   
   ```ssh centos@<new-headnode-ip>```

1. Your cluster is now up and running. You can submit jobs via sbatch or srun. If you
   see issues with running jobs, there are useful logs in
   ``` /var/log/slurm/slurm_elastic```
   and
   ``` /var/log/slurm/slurmctld.log```

1. When you are done, you may destroy your cluster via:

   ```./cluster_destroy.sh -n <headnode-name>```
   
   The optional -v flag will remove your shared volume if one was created- if you'd 
   like to keep that data, do not run this with -v!
   
   

Useage note:
Slurm will run the suspend/resume scripts in response to 

``` bash
scontrol update nodename=compute-[0-1] state=power_down
```
 
or

```bash
scontrol update nodename=compute-[0-1] state=power_up
```

If compute instances get stuck in a bad state, it's often helpful to
cycle through the following:

``` bash
scontrol update nodename=compute-[?] state=down reason=resetting
scontrol update nodename=compute-[?] state=power_down
scontrol update nodename=compute-[?] state=idle
```

or to re-run the suspend/resume scripts as above (if the instance
power state doesn't match the current state as seen by slurm).

This work supported by [![NSF-1548562](https://img.shields.io/badge/NSF-1548562-blue.svg)](https://nsf.gov/awardsearch/showAward?AWD_ID=1548562)
