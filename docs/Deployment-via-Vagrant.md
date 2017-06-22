# Deployment via Vagrant
A [Vagrantfile](../Vagrantfile) is included in this repository that can be used to deploy NGINX locally (to one or more VMs hosted under [VirtualBox](https://www.virtualbox.org/)) using [Vagrant](https://www.vagrantup.com/).  From the top-level directory of this repository a command like the following will (by default) deploy NGINX to a single CentOS 7 virtual machine running under VirtualBox:

```bash
$ vagrant -a="192.168.34.250" up
```

Note that the `-a, --addr-list` flag must be used to pass an IP address (or a comma-separated list of IP addresses) into the [Vagrantfile](../Vagrantfile). In this example, there is only one IP address in that list, so a single-node NGINX deployment will be performed

As was mentioned previously, if we are performing a multi-node NGINX cluster deployment, then we simply need to include a comma-separated list of IP addresses for the nodes that we are deploying as part of the `vagrant ... up` command, as in this example:

```bash
$ vagrant -a="192.168.34.250,192.168.34.251,192.168.34.252" \
    -i="192.168.34.242" up
```

Note that in this command we have also specified a virtual IP address for the cluster to use (via the `-i, --virtual-ip` flag). This is the IP address that the active node will be listening on, and if the active node fails, the `keepalived` daemons on the cluster nodes will work together to ensure that the next node (in terms of priority) steps in to handle user requests.

In terms of how it all works, the [Vagrantfile](../Vagrantfile) is written in such a way that the following sequence of events occurs when the `vagrant ... up` command shown above is run:

1. All of the virtual machines in the cluster (the addresses in the `-a, --addr-list`) are created
1. After all of the nodes have been created, NGINX is deployed all of those nodes
1. If there is more than one node, the `keepalived` package is also deployed to these nodes and they are configured as an active-passive cluster in a single Ansible playbook run
1. The `nginx` service is started on all of the nodes that were just provisioned

Once the playbook run triggered by the [Vagrantfile](../Vagrantfile) is complete, we can browse to the (virtual) IP address of our NGINX node(s) and we should see the default NGINX homepage.

So, to recap, by using a single `vagrant ... up` command we were able to quickly spin up a cluster consisting of of three NGINX nodes and configure those nodes as a an active-passive cluster that managed using `keepalived` and that is listening on a virtual IP address.

## Separating instance creation from provisioning
While the `vagrant up` commands that are shown above can be used to easily deploy NGINX to a single node or to build a multi-node NGINX cluster, the [Vagrantfile](../Vagrantfile) included in this repository also supports separating out the creation of the virtual machine from the provisioning of that virtual machine using the Ansible playbook contained in this repository's [provision-nginx.yml](../provision-nginx.yml) file.

To create a set of virtual machines that we plan on using to build a NGINX cluster without provisioning NGINX to those machines, simply run a command similar to the following:

```bash
$ vagrant -a="192.168.34.250,192.168.34.251,192.168.34.252" \
    up --no-provision
```

This will create a set of three virtual machines with the appropriate IP addresses ("192.168.34.250", "192.168.34.251", and "192.168.34.252"), but will skip the process of provisioning those VMs with an instance of NGINX. Note that when you are creating the virtual machines but skipping the provisioning step it is not necessary to provide the virtual IP address that the cluster will listen on using the `-i, --virtual-ip` flag.

To provision the machines that were created above and configure those machines as a NGINX cluster, we simply need to run a command like the following:

```bash
$ vagrant -a="192.168.34.250,192.168.34.251,192.168.34.252" \
    -i="192.168.34.242" provision
```

That command will attach to the named instances and run the playbook in this repository's [provision-nginx.yml](../provision-nginx.yml) file on those nodes, resulting in a multi-node NGINX cluster consisting of the nodes that were created in the `vagrant ... up --no-provision` command that was shown, above.

## Additional vagrant deployment options
While the commands shown above will install NGINX with a reasonable, default configuration from a standard location, there are additional command-line parameters that can be used to control the deployment process triggered by a `vagrant ... up` or `vagrant ... provision` command. Here is a complete list of the command-line flags that can be supported by the [Vagrantfile](../Vagrantfile) in this repository:

* **`-a, --addr-list`**: the NGINX address list; this is the list of nodes that will be created and provisioned, either by a single `vagrant ... up` command or by a `vagrant ... up --no-provision` command followed by a `vagrant ... provision` command; this command-line flag **must** be provided for almost every `vagrant` command supported by the [Vagrantfile](../Vagrantfile) in this repository
* **`-i, --virtual-ip`**: the virtual IP address that `keepalived` will assign to the active node in the cluster; this argument **must** be provided for any `vagrant` commands that involve provisioning of the instances that make up a multi-node NGINX cluster
* **`-b, --basic-auth`**: if this flag is set on the command-line during provisioning, then the nodes will be configured such that a username and password **must** be provided to access the NGINX instance; the usernames that are automatically given access are defined by the `nginx_admin_user` and `nginx_power_user` parameters, which default to `admin` and `power_user`, respectively; passwords will be automatically created for these users by the playbook run, and those passwords will be saved to the `credentials/{{user}}/password.txt` file (where the `{{user}}` is one of the usernames mentioned above); by default basic authentication is not enabled
* **`-o, --https-only`**: if this flag is set, then the NGINX instance (or instances) will be configured such that any HTTP requests that are received are redirected as HTTPS requests; by default the instance (or instances) will support both HTTP and HTTPS requests
* **`-y, --yum-url`**: the local YUM repository URL that should be used when installing packages during the node provisioning process. This can be useful when installing NGINX onto CentOS-based VMs in situations where there is limited (or no) internet access; in this situation the user might want to install packages from a local YUM repository instead of the standard CentOS mirrors. It should be noted here that this parameter is not used when installing NGINX on RHEL-based VMs; in such VMs this option will be silently ignored if set
* **`-e, --epel-url`**: the local EPEL repository URL that should be used when installing NGINX during the node provisioning process. This can be useful when installing NGINX onto CentOS-based VMs in situations where there is limited (or no) internet access; in this situation the user might want to install NGINX from a local EPEL repository instead of the standard EPEL mirrors.
* **`-c, --clear-proxy-settings`**: if set, this command-line flag will cause the playbook to clear any proxy settings that may have been set on the machines being provisioned in a previous ansible-playbook run. This is useful for situations where an HTTP proxy might have been set incorrectly and, in a subsequent playbook run, we need to clear those settings before attempting to install any packages (for example)
* **`-r, --reset-mirrors`**: if this flag is included on the command line then the YUM and/or EPEL repository files that were modified (using the **`-y, --yum-url`** and/or **`-e, --epel-url`** flags, respectively) will be reset to their original (unmodified) versions by copying the backup files that were created when the YUM and/or EPEL repository files were modified; it should be noted here that:
    * if a **`-y, --yum-url`** and/or **`-e, --epel-url`** flag was never used to setup a local mirror then this flag will have no effect (since the corresponding repository file will not have been modified (and a backup of that file will not exist yet)
    * similarly if a local mirror value is provided as part of the same provisioning command (using the **`-y, --yum-url`** and/or **`-e, --epel-url`** flag), this flag will be silently ignored for those repositories
* **`-f, --local-vars-file`**: the *local variables file* that should be used when deploying the cluster. A local variables file can be used to maintain the configuration parameter definitions that should be used for a given NGINX cluster deployment, and values in this file will override any values that are either embedded in the [vars/nginx.yml](../vars/nginx.yml) file as defaults or passed into the `ansible-playbook` command as extra variables

As an example of how these options might be used, the following command will create a three-node NGINX cluster with one active node listening at the IP address `192.168.34.242`, with access to that instance controlled by basic authentication and with the nodes configured such that any HTTP requests are redirected as HTTPS requests when provisioning the machines created by the `vagrant ... up --no-provision` command shown above:

```bash
$ vagrant -a="192.168.34.250,192.168.34.251,192.168.34.252" \
    -i="192.168.34.242" -b -o provision
```

While the list of command-line options defined above may not cover all of the customizations that user might want to make when performing production NGINX deployments, our hope is that the list shown above is more than sufficient for deploying NGINX clusters using Vagrant for local testing purposes.
