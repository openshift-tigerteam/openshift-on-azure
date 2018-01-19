# Installing up OpenShift on Azure

### Set up Hosts (if not done before)
**If using a jump server other than master and if you already provisioned your VMs you can ignore this stage and move to the next one **

Spin up a master and a few node hosts on the AWS environment  

* 2 cores and 8GB RAM or higher
* 2 extra disks on the master
  *  30GB or higher for docker storage
  *  30GB or higher for registry storage
* Master needs a public IP
* Configure a security group for the master that allows
 * TCP SSH port 22
 * HTTP port 80
 * HTTPS port 443
 * TCP port 8443 for MasterAPI
 * TCP port 9090 for Cockpit
 * TCP port range 2379-2380
* Set up your sshkey to be able to log into the master


We will use master as our jump host to install OpenShift using Ansible.

*  Log into the master
* `sudo bash` and then as root user subscribe to RHN
* install `atomic-openshift-utils` and this will install ansible

 ```
 subscription-manager register
 subscription-manager attach --pool <<your poolid>>
 subscription-manager repos --disable="*"
 subscription-manager repos     --enable="rhel-7-server-rpms"     --enable="rhel-7-server-extras-rpms"     --enable="rhel-7-server-ose-3.7-rpms"
 yum install -y atomic-openshift-utils
```

* Switch back to regular user

	```
	ssh-keygen
	```

* Use this key (`cat ~/.ssh/id_rsa.pub` as the login key for the other hosts. Configure this from Azure console  Node->Support+Troubleshooting->Reset Password

	Now you should be able to ssh from master host to the other (node) hosts.

* Install git
```yum install git -y ```


* `git clone` the repository (https://github.com/VeerMuchandi/openshift-on-azure) onto the master host. For now using context-dir v3.3. You should now get the required ansible playbooks to prep the hosts

### Prepare the Hosts

* Update the `hosts.openshiftprep` file with the internal ip addresses of all the hosts (master and the node hosts). In my case these were `10.0.0.5, 10.0.0.6 10.0.0.7, 10.0.0.8, 10.0.0.9 and 10.0.0.10`

* Update the `openshifthostprep.yml` file to point the variable `docker_storage_mount` to where ever your extra-storage was mounted. In my case, it was `/dev/sdc`. You can find this by running `fdisk -l`

* Run the playbook to prepare the hosts.  

```
ansible-playbook -i hosts.openshiftprep openshifthostprep.yml
```

**Configure storage server**

Edit the hosts.storage file to include the master's hostname/ip and run the playbook that configures storage

```
ansible-playbook -i hosts.storage configure-storage.yml
```

### Add DNS entries

If you have an external DNS server, make the

*  A record entries for the master url to point to the IP address of the host
*  Wild card DNS entry/entries to point to the hosts where Router would run

```
A	master.devday	40.112.62.165	1 Hour	Edit
A	*.apps.devday	40.112.62.166	1 Hour	Edit
A	*.apps.devday	40.112.62.167	1 Hour	Edit
```


### Run OpenShift Installer

Edit /etc/ansible/hosts file
* This config is for installing master, infra-nodes and nodes
* Router, Registry and Metrics will be installed automatically
* It also sets up a server as NFS server. This is where we configured extra storage as `/exports`. This playbook will create PVs for registry and metrics and uses them as storage
* Deploys redundant registry and router

```
# Create an OSEv3 group that contains the masters and nodes groups
[OSEv3:children]
masters
nodes
nfs

# Set variables common for all OSEv3 hosts
[OSEv3:vars]
# SSH user, this user should allow ssh based auth without requiring a password
ansible_ssh_user=ocpadmin

# If ansible_ssh_user is not root, ansible_sudo must be set to true
ansible_sudo=true
ansible_become=yes

# To deploy origin, change deployment_type to origin
deployment_type=openshift-enterprise

openshift_clock_enabled=true

#os_sdn_network_plugin_name=redhat/openshift-ovs-multitenant
#os_sdn_network_plugin_name=redhat/openshift-ovs-networkpolicy

# Disabling for smaller instances used for Demo purposes. Use instances with minimum disk and memory sizes required by OpenShift
openshift_disable_check=disk_availability,memory_availability,docker_image_availability,docker_storage

openshift_master_default_subdomain=apps.cgpoc.openshift.online
openshift_hosted_router_selector='region=infra'
openshift_registry_selector='region=infra'
openshift_install_examples=true

## The two parameters below would be used if you want API Server and Master running on 443 instead of 8443.
## In this cluster 443 is used by router, so we cannot use 443 for master
#openshift_master_api_port=443
#openshift_master_console_port=443

openshift_hosted_registry_storage_nfs_directory=/exports

# Enable cluster metrics
openshift_metrics_install_metrics=true
openshift_metrics_storage_kind=nfs
openshift_metrics_storage_access_modes=['ReadWriteOnce']
openshift_metrics_storage_nfs_directory=/exports
openshift_metrics_storage_nfs_options='*(rw,root_squash)'
openshift_metrics_storage_volume_name=metrics
openshift_metrics_storage_volume_size=20Gi
openshift_metrics_storage_labels={'storage': 'metrics'}

# Logging
openshift_logging_install_logging=true
openshift_logging_storage_kind=nfs
openshift_logging_storage_access_modes=['ReadWriteOnce']
openshift_logging_storage_nfs_directory=/exports
openshift_logging_storage_nfs_options='*(rw,root_squash)'
openshift_logging_storage_volume_name=logging-es
openshift_logging_storage_volume_size=20Gi
openshift_logging_storage_labels={'storage': 'logging'}
openshift_logging_es_cluster_size=1
openshift_logging_es_memory_limit=2Gi


# Registry
openshift_hosted_registry_storage_kind=nfs
openshift_hosted_registry_storage_access_modes=['ReadWriteMany']
openshift_hosted_registry_storage_nfs_directory=/exports
openshift_hosted_registry_storage_nfs_options='*(rw,root_squash)'
openshift_hosted_registry_storage_volume_name=registry
openshift_hosted_registry_storage_volume_size=20Gi

# enable htpasswd authentication
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/openshift/openshift-passwd'}]

# host group for masters
[masters]
10.0.0.4

[nfs]
10.0.0.4

[etcd]
10.0.0.4

# host group for nodes, includes region info
[nodes]
10.0.0.4 openshift_ip=10.0.0.4 openshift_hostname=ocpmaster openshift_public_ip=138.91.197.127 openshift_node_labels="{'region': 'infra', 'zone': 'default'}"  openshift_scheduleable=true openshift_public_hostname=ocpmaster.cgpoc.openshift.online
10.0.0.5 openshift_ip=10.0.0.5 openshift_hostname=ocpnode1 openshift_public_hostname=node01.cgpoc.openshift.online openshift_node_labels="{'region': 'primary', 'zone': 'west'}"
10.0.0.6 openshift_ip=10.0.0.6 openshift_hostname=ocpnode2 openshift_public_hostname=node01.cgpoc.openshift.online openshift_node_labels="{'region': 'primary', 'zone': 'east'}"
10.0.0.7 openshift_ip=10.0.0.7 openshift_hostname=ocpnode3 openshift_public_hostname=node01.cgpoc.openshift.online openshift_node_labels="{'region': 'primary', 'zone': 'south'}"


```

Now you need to run the following to make sure NetworkManager working correctly
echo $(date) " - Running network_manager.yml playbook"

```
DOMAIN=`domainname -d`
```

Setup NetworkManager to manage eth0

```
ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/byo/openshift-node/network_manager.yml

```

Configure resolv.conf on all hosts through NetworkManager

```
ansible all -m shell -a "systemctl restart NetworkManager"
ansible all -m shell -a "nmcli con modify eth0 ipv4.dns-search $DOMAIN"
ansible all -m shell -a "systemctl restart NetworkManager"

```

Verify /etc/resolv.conf file has the correct domain information before starting the installation.

Now run the OpenShift ansible installer

```
ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/byo/config.yml
```

Note that registry, router and metrics will all be installed.

### Add Users
Once OpenShift is installed add users

```
touch /etc/openshift/openshift-passwd
htpasswd /etc/openshift/openshift-passwd <<uname>>

```

### Post Installation
In OCP 3.3 there are certain tech preview features. Pipelines is one of them and is not enabled by default. Following steps will enable the same. I believe this will be temporary step until the tech preview becomes supported.

* Ensure master host is listed in `hosts.master`
* Run the post-install script that will enable pipelines feature

```
ansible-playbook -i hosts.master post-install.yml
```
### Adding Azure Dynamic storage
1. create /etc/azure/azure.conf file on all nodes (masters and nodes)
example here:
{"resourceGroup": "ocp37", "subscriptionID": "68acxxxx-xxxx-xxxx-xxxx-f2a7a6c0xxxx", "aadClientID": "535axxxx-xxxx-xxxx-xxxx-eaa74833xxxx", "aadClientSecret": "xxxxxxxx", "tenantID": "ce1cxxxx-xxxx-xxxx-xxxx-b7a87145xxxx"}

2. On master, update /etc/origin/master/master-config.yaml

```
...
kubernetesMasterConfig:
  apiServerArguments:
    cloud-config:
    - /etc/azure/azure.conf
    cloud-provider:
    - azure
    runtime-config:
    - apis/settings.k8s.io/v1alpha1=true
    storage-backend:
    - etcd3
    storage-media-type:
    - application/vnd.kubernetes.protobuf
  controllerArguments:
    cloud-provider:
      - azure
    cloud-config:
      - /etc/azure/azure.conf
...
```
Restart master services

```
systemctl restart atomic-openshift-master-api
systemctl restart atomic-openshift-master-controllers

```
you can watch the log by running:

```
journalctl -u atomic-openshift-master-controllers -f

```


3. On all nodes, update /etc/origin/node/node-config.yaml

```
...
kubeletArguments:
  cloud-config:
  - /etc/azure/azure.conf
  cloud-provider:
  - azure
...
```
Restart node service

```
systemctl restart atomic-openshift-node

```
you can watch the log by running:

```
journalctl -u atomic-openshift-node -f

```

If you are running `oc get nodes` while restarting the services, you will see the node get deleted and added back into the list. Give it time for node to add back it.

4. Adding azure storage class by creating the file below.

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: azure-storageclass
annotations:
  storageclass.beta.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/azure-disk
parameters:
  storageAccount: xxxxxxxx
  location: westus
```

5. Set Azure storageclass as default storageclass

```
oc patch storageclass azure-storageclass \
-p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "false"}}}'

```

###Reset Nodes

If you ever want to clean up docker storage and reset the node(s):

1. Update the `hosts.nodereset` file to include the list of hosts to reset.

2. Run the playbook to reset the node
```
$ ansible-playbook -i hosts.nodereset node-reset.yml
```
