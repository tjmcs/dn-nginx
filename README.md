# dn-nginx
Playbooks/Roles used to deploy NGINX; supports either single-node or multi-node NGINX deployments.

# Installation
To install NGINX using the [provision-nginx.yml](provision-nginx.yml) playbook in this repository, first clone the contents of this repository to a local directory using a command like the following:

```bash
$ git clone --recursive https://github.com/Datanexus/dn-nginx
```

That command will pull down the repository and it's dependencies. Currently this playbook's only dependencies are on the [common-roles](https://github.com/Datanexus/common-roles), [common-flows](https://github.com/Datanexus/common-flows), and [common-utils](https://github.com/Datanexus/common-utils) submodules in this repository. The first provides a set of common roles that are reused across the DataNexus playbooks, while the second provides a similar set "flows" that are common across the DataNexus playbooks, and the third provides a set of common utilities, including the of dynamic inventory script that is used to control deployments made using this playbook in OpenStack environments.

The only other requirements for using the playbook in this repository are a relatively recent (v2.x) release of Ansible. The easiest way to obtain a recent relese if Ansible is via a `pip install`, which requires that Python and pip are both installed locally. We have performed all of our testing using a recent (2.7.x) version of Python (Python 2); your mileage may vary if you attempt to run the playbook or the attached dynamic inventory scripts under a newer (v3.x) release of Python (Python 3).

# Using this role to deploy NGINX
The [provision-nginx.yml](provision-nginx.yml) file at the top-level of this repository supports both single-node NGINX deployments and the deployment of multi-node NGINX clusters (where the nodes of the cluster are maintained in an active-passive configuration using `keepalived`). The process of deploying NGINX to these nodes will vary, depending on whether you are managing your inventory dynamically or statically (more on this topic [here](docs/Dynamic-vs-Static-Inventory.md)), whether you are performing a single-node deployment or are deploying a multi-hnode NGINX cluster, and where you are downloading the packages and dependencies from that are needed to run NGINX on those nodes.

We discuss the various deployment scenarios supported by this playbook in [this document](docs/Deployment-Scenarios.md) and discuss how the [Vagrantfile](Vagrantfile) in this repository can be used to deploy NGINX (both single-node deployments and multi-node clusters are supported) to a set of VMs hosted locally in VirtualBox [here](docs/Deployment-via-Vagrant.md).

## Controlling the configuration
This repository includes a default set of parameters defined in the [vars/nginx.yml](vars/nginx.yml) file that make it possible to perform deployments of NGINX out of the box with few, if any, changes necessary. If you are not happy with the default configuration defined in that file, there are a number of ways that you can customize the configuration used for your deployment, and which method you use is entirely up to you:

* You can edit the [vars/nginx.yml](vars/nginx.yml) file to modify the default values that are defined in that file or define additional configuration parameters
* You can override the values defined in that file or define additional configuration parameters by passing the values for those parameters into your `ansible-playbook` run on the command-line as extra variables
* You can setup a *local variables file* on the local filesystem of the Ansible host that contains the values for the parameters you wish to set or customize, then pass the location of that file into your `ansible-playbook` command as an extra variable

We have provided a summary of the configuration parameters that can be set (using any of these three methods) during an `ansible-playbook` run [here](docs/Supported-Config-Params.md). Overall, we have found the last option to be the easiest and most flexible of those three options. This is because:

* It avoids modifying files that are being tracked under version control in the main, `dn-nginx` repository (the first option); making such changes will, more than likely, lead to conflicts at a later date when these files are modified in the main `dn-nginx` repository in a way that is inconsistent with the values that you have set in your clone, locally.
* It lets you maintain your preferred configuration for any given NGINX deployment in the form of a configuration file, which you can easily maintain (along with the configuration files used for other deployments you have made) under version control in a separate repository
* It provides a record of the configuration of any given deployment, which is in direct contrast to the second option (where the configuration parameters for any given deployment are passed in on the command-line as extra variables)

That being said, the second option may be useful for some deployment scenarios (a one-off deployment of a local test environment, for example), so it remains a viable option for some users. Overall, we would recommend against trying to maintain your preferred cluster configuration by modifying the values defined in the [vars/nginx.yml](vars/nginx.yml) file.

# Assumptions
It is assumed that this playbook will be run on a recent (systemd-based) version of RHEL or CentOS (RHEL-7.x or CentOS-7.x, for example); no support is provided for other distributions (and the `provision-nginx.yml` playbook will not run successfully). Furthermore, it is assumed that you are interested in deploying a relatively recent version of NGINX using this playbook (the current default is to deploy the latest version available in the EPEL repositories).

It should also be noted that in order to execute the vagrant commands shown in [this document](docs/Deployment-via-Vagrant.md) locally, recent versions of [Vagrant](https://www.vagrantup.com/) and [VirtualBox](https://www.virtualbox.org) will have to be installed locally. While Vagrant does support management of Virtual Machines deployed via VMware Workstation and/or Fusion with the right (commercial) drivers in place, we have only tested the [Vagrantfile](Vagrantfile) in this repository under VirtualBox using recent (v1.9.x) releases of Vagrant.
