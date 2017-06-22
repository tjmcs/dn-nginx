# Supported configuration parameters
The playbook in the [provision-nginx.yml](../provision-nginx.yml) file in this repository pulls in a set of default values for many of the configuration parameters that are needed to deploy NGINX from the [vars/nginx.yml](../vars/nginx.yml) file. The parameters defined in that file define a reasonable set of defaults for a fairly generic NGINX deployment, either to a single node or a cluster, including defaults for the whether the instances should be secured or not using basic authentication (by default they are not secured), whether the instances should support HTTP requests or not (by default they do), the interface they should listen on for client connections (`eth0` by default), and the packages that must be installed on the node before the `nginx` service can be started.

In addition to the defaults defined in the [vars/nginx.yml](../vars/nginx.yml) file, there are a larger set of parameters that can be used to either control the deployment of NGINX to the nodes that will make up a cluster during an `ansible-playbook` run or to configure those NGINX nodes once the installation is complete. In this section, we summarize these options, breaking them out into:

* parameters used to control the `ansible-playbook` run
* parameters used during the deployment process itself, and
* parameters used to configure our NGINX nodes once NGINX has been installed locally.

Each of these sets of parameters are described in their own section, below.

## Parameters used to control the playbook run
The following parameters can be used to control the `ansible-playbook` run itself, defining things like how Ansible should connect to the nodes involved in the playbook run, which nodes should be targeted, where the NGINX distribution should be downloaded from, which packages must be installed during the deployment process, and where those packages should be obtained from:

* **`ansible_ssh_private_key_file`**: the location of the private key that should be used when connecting to the target nodes via SSH; this parameter is useful when there is one private key that is used to connect to all of the target nodes in a given playbook run
* **`ansible_user`**: the username that should be used when connecting to the target nodes via SSH; is useful if the same username is used when connecting to all of the target nodes in a given playbook run
* **`cloud`**: if the inventory is being managed dynamically, this parameter is used to indicate the type of target cloud for the deployment (either `aws` or `osp`); this controls how the [build-app-host-groups](../common-roles/build-app-host-groups) common role retrieves the list of target nodes for the deployment
* **`private_key_path`**: used to define the directory where the private keys are maintained when the inventory for the playbook run is being managed dynamically; in these cases, the scripts used to retrieve the dynamic inventory information will return the names of the keys that should be used to access each node, and the playbook will search the directory specified by this parameter to find the corresponding key files. If this value is not specified then the current working directory will be searched for those keys by default
* **`proxy_env`**: a hash map that is used to define the proxy settings to use for downloading distribution files and installing packages; supports the `http_proxy`, `no_proxy`, `proxy_username`, and `proxy_password` fields as part of this hash map
* **`reset_proxy_settings`**: used to reset any HTTP/YUM proxy settings that may have been made in a previous playbook run back to the defaults (no proxy); this is useful when a proxy was incorrectly set in a previous playbook run and the user wants to return to a "no-proxy" setup in the current playbook run
* **`reset_mirrors`**: used to reset any YUM/EPEL repository files that may have been modified in a previous playbook run back to their default (not modified) state; this is useful when a mirror was incorrectly set in a previous playbook run and the user wants to return to a "no-miror" setup in the current playbook run
* **`yum_repo_url`**: used to set the URL for a local YUM mirror. This parameter is only used for CentOS-based deployments; when deploying NGINX to RHEL-based nodes this parameter is silently ignored and the RHEL package repositories defined locally on the node will be used for any packages installed during the deployment process
* **`epel_repo_url`**: used to set the URL for a local EPEL mirror.

## Parameters used during the deployment process
These parameters are used to control the deployment process itself, defining things like which packages to install.

* **`nginx_package_list`**: the list of packages that should be installed on the NGINX nodes; typically this parameter is left unchanged from the default (which installs the `epel-release` and `httpd-tools` packages), but if it is modified the default, these two packages must be included as part of the new package list or an error will result when attempting to install and configure the `nginx` package (the first is used to configure the nodes so that the `nginx` package is available; the second is used to setup privately signed certificates that are needed to support HTTPS requests)

## Parameters used to configure the NGINX nodes
These parameters are used configure the NGINX nodes themselves during a playbook run, defining things like the interfaces that NGINX should be listening on for requests and the directory where NGINX should store its data.

* **`data_iface`**: the name of the interface that the NGINX nodes in the cluster should use when talking with each other and when talking to the Zookeeper ensemble. This interface typically corresponds to a private or management network, with no customer access. An interface of this name must exist for the playbook to run successfully, and if unspecified a value of `eth0` is assumed
* **`api_iface`**: the name of the interface that the NGINX nodes should listen on for user requests. This network corresponds to a public network since customers will use this interface to access the data in the NGINX cluster. An interface of this name must exist for the playbook to run successfully, and if unspecified a value of `eth0` is assumed
* **`nginx_virtual_ip`**: the virtual IP address that the active NGINX instance will be configured to listen on in an active-passive NGINX cluster; this parameter **must** be specified for multi-node NGINX deployments
* **`iface_description_array`**: this parameter can be used in place of the `data_iface` and `api_iface` parameters described above, and it provides users with the ability to specify a description of these two interfaces rather than identifying them by name (more on this, below)
* **`nginx_admin_user`**: the name to use for the *administrative user* account that is created during provisioning when basic authentication is enabled; defaults to `admin`
* **`nginx_power_user`**: the name to use when constructing an *power user* account that is created during provisioning when basic authentication is enabled; defaults to `power_user`
* **`nginx_basic_auth`**: if this flag is set to `true`, then basic authentication will be enabled on all of the NGINX instances targeted by the deployment for access control
    * in this scenario, a password will be generated for the **`nginx_admin_user`** and **`nginx_power_user`**, respectively (see above)
    * those auto-generated passwords will be saved in the `credentials/{{user}}/password.txt` file under the directory where the `ansible-playbook` command is run (note that in this example, the string `{{user}}` represents one of the usernames being setup, eg. `admin` or `power_user`); defaults to `false`
* **`nginx_https_only`**: if this flag is set to `true`, then any HTTP requests received will be redirected as HTTPS requests; defaults to `false`
* **`nginx_country`**: the country used when constructing the self-signed certificates needed to support HTTPS requests; defaults to `US` if not specified
* **`nginx_state`**: the state used when constructing the self-signed certificates needed to support HTTPS requests; defaults to `CO` if not specified
* **`nginx_location`**: the location used when constructing the self-signed certificates needed to support HTTPS requests; defaults to `Denver` if not specified
* **`nginx_org`**: ; the organization used when constructing the self-signed certificates needed to support HTTPS requests; defaults to `IT` if not specified

## Interface names vs. interface descriptions
For some operating systems on some platforms, it can be difficult (if not impossible) to determine the names of the interfaces that should be passed into the playbook using the `data_iface` and `api_iface` parameters that we described, above. In those situations, the playbook in this repository provides an alternative; specifying those interfaces using the `iface_description_array` parameter instead.

Put quite simply, the `iface_description_array` lets you specify a description for each of the networks that you are interested in, then retrieve the names of those networks on each machine in a variable that can be used elsewhere in the playbook. To accomplish this, the `iface_description_array` is defined as an array of hashes (one per interface), each of which include the following fields:

* **`type`**: the type of description being provided, currently only the `cidr` type is supported
* **`val`**: a value describing the network in question; since only `cidr` descriptions are currently supported, a CIDR value that looks something like `192.168.34.0/24` should be used for this field
* **`as_var`**: the name of the variable that you would like the interface name returned as

With these values in hand, the playbook will search the available networks on each machine and return a list of the interface names for each network that was described in the `iface_description_array` as the value of the fact named in the `as_var` field for that network's entry. For example, given this description:

```json
    iface_description_array: [
        { as_var: 'data_iface', type: 'cidr', val: '192.168.34.0/24' },
        { as_var: 'api_iface', type: 'cidr', val: '192.168.44.0/24' },
    ]
```

the playbook will return the name of the network that matches the CIDR `192.168.34.0/24` as the value of the `data_iface` fact and the name of the network that matches the CIDR `192.168.34.0/24` as the value of the `api_iface` fact. These two facts can then be used later in the playbook to correctly configure the nodes to talk to each other and listen on the proper interfaces for user requests.

It should be noted that if you choose to take this approach when constructing your `ansible-playbook` runs, a matching entry in the `iface_description_array` must be specified for both the `data_iface` and `api_iface` networks, otherwise the default value of `eth0` will be used for these facts (and the playbook run may result in nodes that are at best misconfigured; if the `eth0` network does not exist then the playbook will fail to run altogether).
