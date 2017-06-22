# Example deployment scenarios

There are a three basic deployment scenarios that are supported by this playbook. In the first two scenarios (shown below) we'll walk through the deployment of NGINX to a single node and the deployment of a multi-node NGINX cluster using a static inventory file. Finally, in the third scenario, we will show how the same multi-node NGINX cluster deployment shown in the second scenario could be performed using the dynamic inventory scripts for both AWS and OpenStack instead of a static inventory file.

## Scenario #1: deploying NGINX to a single node
While this is the simplest of the deployment scenarios that are supported by this playbook, it is more than likely that deployment of NGINX to a single node is really only only useful non-production environments. Nevertheless, we will start our discussion with this deployment scenario since it is the simplest.

If we want to deploy NGINX to a single node with the IP address "192.168.34.250", we would start by creating a very simple inventory file that looks something like the following:

```bash
$ cat single-node-inventory

192.168.34.250 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_node_private_key'

[nginx]
192.168.34.250

$ 
```

Note that in this inventory file the `ansible_ssh_host` and `ansible_ssh_port` parameters will take their default values since they aren't specified for the single host entry in the inventory file.

To deploy the NGINX to our single node, we would simply run an `ansible-playbook` command that looks something like this:

```bash
$ ansible-playbook -i single-node-inventory provision-nginx.yml
```

This will install NGINX as packages from the EPEL repository and configure the node as a single-node NGINX instance (using the default configuration parameters that are defined in the [vars/nginx.yml](../vars/nginx.yml) file).

## Scenario #2: deploying a multi-node NGINX cluster
If you are using this playbook to deploy a multi-node NGINX cluster, then the configuration is a bit more complicated. The playbook assumes that if you are deploying a multi-node NGINX cluster, you would also like to ensure that the cluster is relatively tolerant of the failure of a node. This is accomplished by configuring the nodes in the NGINX cluster that is being deployed in an active-passive configuration, with `keepalived` deployed to all of the nodes in the cluster and one node (the active node) listening on a virtual IP address that is managed by the `keepalived` daemon. If the `nginx` service fails on the active node (or if the active node fails altogether), then the `keepalived` daemons on the other nodes will work together to select another node (based on the priority in each node's configuration) to take over as the active node. If the node that failed comes back (or the `nginx` service on that node is restarted or recovers), then that node will once again become the active node and the node that took over during the outage of the primary node will return to a passive state.

So, assuming that we want to deploy a three node NGINX cluster, let's walk through what the commands that we'll need to run look like. In addition, let's assume that we're going to be using a static inventory file to control our NGINX deployment. The static inventory file that we will be using for this example looks like this:

```bash
$ cat combined-inventory
# example inventory file for a clustered deployment

192.168.34.250 ansible_ssh_host= 192.168.34.250 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/nginx_cluster_private_key'
192.168.34.251 ansible_ssh_host= 192.168.34.251 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/nginx_cluster_private_key'
192.168.34.252 ansible_ssh_host= 192.168.34.252 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/nginx_cluster_private_key'

[nginx]
192.168.34.250
192.168.34.251
192.168.34.252

$
```

With our static inventory file in place, we **must** select a virtual IP address for the active node to listen on, and that IP address **must** be an IP address that is routable from the nodes that make up the cluster. This virtual IP address must be specified as part of our `ansible-playbook` run by providing a value for the `nginx_virtual_ip` parameter (which can be passed in on the command-line as an extra variable or passed in as part of a *local variables file*; we'll describe this in more detail, below).

So, assuming that we've decided to use `192.168.34.242` as our virtual IP address, the command we would use to deploy NGINX to the three nodes in our static inventory file would look something like this:

```bash
$ ansible-playbook -i combined-inventory -e "{ \
      data_iface: eth0, api_iface: eth1, \
      nginx_virtual_ip: 192.168.34.242,
      yum_repo_url: 'http://192.168.34.254/centos' \
    }" provision-nginx.yml
```

Alternatively, rather than passing all of those arguments in on the command-line as extra variables, we can make use of the *local variables file* support that is built into this playbook and construct a YAML file that looks something like this containing the configuration parameters that are being used for this deployment:

```yaml
data_iface: eth0
api_iface: eth1
nginx_virtual_ip: 192.168.34.242
yum_repo_url: 'http://192.168.34.254/centos'
```

and then we can pass in the *local variables file* as an argument to the `ansible-playbook` command; assuming the YAML file shown above was in the current working directory and was named `test-cluster-deployment-params.yml`, the resulting command would look something like this:

```bash
$ ansible-playbook -i combined-inventory -e "{ \
      local_vars_file: 'test-cluster-deployment-params.yml' \
    }" provision-nginx.yml
```

As an aside, it should be noted here that the [provision-nginx.yml](../provision-nginx.yml) playbook includes a [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) line at the beginning of the playbook file. As such, the playbook can be executed directly as a shell script (rather than using the file as the final input to an `ansible-playbook` command). This means that the command that was just shown could also be run as:

```bash
$ ./provision-nginx.yml -i combined-inventory -e "{ \
      local_vars_file: 'test-cluster-deployment-params.yml' \
    }"
```

This form is available as a replacement for any of the `ansible-playbook` commands that we show here; which form you use will likely be a matter of personal preference (since both accomplish the same thing).

Once the playbook run is complete, we can browse to the virtual IP address we mentioned previously (`http://192.168.34.242`), and we should see the default NGINX home page displayed in the browser window.

In closing, this section showed how a single `ansible-playbook` command could be used to deploy a three-node NGINX cluster from a static inventory file, provided that a virtual IP address was specified. In the next section, we'll show how this same deployment could be accomplished using a dynamic, rather than static, iventory. 

## Scenario #3: deploying a multi-node NGINX cluster via dynamic inventory
In this section we will repeat the multi-node cluster deployment that we just showed in the previous scenario, but we will use the `build-app-host-groups` role that is provided in the [common-roles](../common-roles) submodule to control the deployment of our NGINX cluster in an AWS or OpenStack environment rather than relying on a static inventory file.

To accomplish this, the we have to:

* Tag the instances in the AWS or OpenStack environment that we will be configuring as a multi-node NGINX cluster with the appropriate `Tenant`, `Project`, `Domain`, and `Application` tags. Note that for all of the nodes we are targeting in our playbook runs, we will assign an `Application` tag of `nginx`
* Once all of the nodes that will make up our cluster had been tagged appropriately, we can run `ansible-playbook` command similar to the `ansible-playbook` command shown in the previous scenario; this will deploy NGINX to the cluster nodes and configure them as an active-passive cluster that is managed using `keepalived`

In terms of what the commands look like, lets assume for this example that we've tagged our seed nodes with the following VM tags:

* **Tenant**: labs
* **Project**: projectx
* **Domain**: preprod
* **Application**: nginx

The `ansible-playbook` command used to deploy NGINX to our nodes and configure them as a cluster in an OpenStack environment would then look something like this:

```bash
$ ansible-playbook -e "{ \
        application: nginx, cloud: osp, \
        tenant: labs, project: projectx, domain: preprod, \
        private_key_path: './keys', data_iface: eth0, api_iface: eth1 \
    }" provision-nginx.yml
```

The playbook then uses the tags in this command to identify the target nodes for the playbook run, installs NGINX on those nodes, and configures them as a new NGINX cluster. 

In an AWS environment, the command would look something like this:

```bash
$ AWS_PROFILE=datanexus_west ansible-playbook -e "{ \
        application: nginx, cloud: aws, \
        tenant: labs, project: projectx, domain: preprod, \
        private_key_path: './keys', data_iface: eth0, api_iface: eth1 \
    }" provision-nginx.yml
```

As you can see, these two commands only in terms of the environment variable defined at the beginning of the command-line used to provision to the AWS environment (`AWS_PROFILE=datanexus_west`) and the value defined for the `cloud` variable (`osp` versus `aws`). In both cases the result would be a set of nodes deployed as a multi-node NGINX cluster. The number of nodes in the cluster will be determined (completely) by the number of nodes in the OpenStack or AWS environment that have been tagged with a matching set of `application`, `tenant`, `project` and `domain` tags.
