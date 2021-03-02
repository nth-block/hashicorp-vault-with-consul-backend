# Steps to follow this guide

## Consul server cluster installation

Follow the steps outlined in [01-consul-server-setup.md](01-consul-server-setup.md).

> The consul server setup needs to be done for each consul server that you are using in the cluster.

The template for the connfiguration is as follows. Details about the nuances to remember will follow the config sample.

```
{
  "server": true,
  "node_name": "$NODE_NAME",
  "datacenter": "dc1",
  "data_dir": "$CONSUL_DATA_PATH",
  "bind_addr": "0.0.0.0",
  "client_addr": "0.0.0.0",
  "advertise_addr": "$ADVERTISE_ADDR",
  "bootstrap_expect": $NUMBER_OF_MACHINES_IN_CONSUL_CLUSTER,
  "retry_join": ["$JOIN1", "$JOIN2", "$JOIN3"],
  "ui": true,
  "log_level": "DEBUG",
  "enable_syslog": true,
  "acl_enforce_version_8": false
}
```

The parameters that you must change are as follows
|Parameter|Comment|
|:-------:|:------|
|`server`| This must be set to `true` in all the consul server config files. You will notice that for the vault VM (which is a consul client, we set this if `false` so that the consul instance there acts as client)| 
|`node_name`| Can be any string value. This is used as an identifier within the consul cluster|
|`data_dir`| The directory where the consul data is stored. Ensure that the directory is writeable by the user account under which consul service will be run. In the [01-consul-server-setup.md](01-consul-server-setup.md) file, this requirement is taken care of.|
|`advertise_addr`|This is the IP address of the machine.<p><p>*TODO*: Need to ensure that internal IP addresses in the *lightning stack* are statically linked to the VM|
|`bootstrap_expect`| This must be set to the number of machines that will form the consul cluster|
|`retry_join`| Addresses of all the consul machines that are in the cluster. These are the machines the consul cluster will try to connect to become a part of the culter.<p><p>Therefore, the TODO item in the `advertise_addr` section where we want *static* IPs for the VMs<p><p> **NOTE: THE NUMBER IN THE `bootstrap_expect` PARAMETER AND THE NUMBER OF IP ADDRESSES IN THE LIST FOR `retry_join` MUST MATCH.**|

#

## Vault and consul agent installation

The vault and the consul must be installed on the same machine to enable the vault to hand off the storage tasks to the consul agent.

This configuration seems a little confusing but is easy to understand when broken down into pieces.

Essentially, this VM has 2 services

1. The `vault` services that offers the front-end to the secrets mangement solution.
2. The `consul agent` that offers the secrets storage service

The `consul agent` is configured to talk to the consul server and the `vault` is configured with the `consul agent` as the storage backend.

Therefore, we first have to install the `consul agent` service in the machine first, followed by the installation of the `vault` service.

### Installation of the consul agent

Follow the [02-vault-and-consul-agents-setup.md](02-vault-and-consul-agents-setup.md) file for the installation of the vault.

The template for the connfiguration is as follows. Details about the nuances to remember will follow the config sample.

```
{
  "server": false,
  "datacenter": "dc1",
  "node_name": "$NODE_NAME",
  "data_dir": "$CONSUL_DATA_PATH",
  "bind_addr": "$BIND_ADDR",
  "client_addr": "127.0.0.1",
  "retry_join": ["$JOIN1", "$JOIN2", "$JOIN3"],
  "log_level": "DEBUG",
  "enable_syslog": true,
  "acl_enforce_version_8": false
}
```

> Crucial things to note here are as follows

|Parameter|Description|
|:-------:|:----------|
|`server`| This is set here to `false` as this machine is a consul client|
|`bind_addr`|Must be set to the IP address of the VM. In other words, the IP from the `api_addr` configuration of the vault server must be used here.|
|`retry_join`| Must be a replica of the entry in the server. I.E. if the server has one member, this must contain only one ip address, if you are using a two node consul cluster, then this must the IP address of both the servers.|

> An astute eye would have also caught the fact that in the server we are configuring the UI while in the client we are not doing the same. The idea is to **disable** the UI in production instance of the consul server cluster too.

### Installation of the vault

The vault configuration is again outlined in the [02-vault-and-consul-agents-setup.md](02-vault-and-consul-agents-setup.md) file.

Config file sample for the vault configuration

```
listener "tcp" {
  address          = "0.0.0.0:8200"
  cluster_address  = "10.0.0.6:8201"
  tls_disable      = "true"
}

storage "consul" {
  address = "127.0.0.1:8500"
  path    = "vault/"
}

ui = true
api_addr =  "http://10.0.0.6:8200"
cluster_addr = "https://10.0.0.6:8201"
```

> Nevermind the `cluster_addr` is configured to use a https endpoint. We are not using that in a single vault instance setup

> *Futhermore, we are also not using the TLS endpoints for the vault API endpoint. **This must be avoided in production.***
 

# Demystifying consul backend for vault

Consul is [Service Mesh](https://en.wikipedia.org/wiki/Service_mesh) solution by Hashicorp and can be used independently of vault for delivering [Microservices applications](https://en.wikipedia.org/wiki/Microservices).

Furthermore, the consul solution also offers the ability to act as a storage backend for vault. Essentially, the vault is a front-end service offering the interface for secrets management and the consul offers the storage area. The *service mesh* capabiltiies of consul are leveraged by vault (which has consul client installed) to ***discover*** the storage backend for storing secrets.

Over the course of operation, the vault is configured to offload the task of finding the storage to the consul agent that is running the vault VM. When the vault gets a request to store or retrieve secrets, the vault offloads the storage lookup task to the consul agent which in turn initates a dialog with the consul server for storage of the secret.

Presumably, the consul server says, "Hang on just a sec! I will store the secret/or fetch it for you and you are good to go", going on to store or retrieve the secret.

# Open TODOs

[] The systemd services that are defined do not start automatically on a system reboot. This support must be added.
[] Static IP addresses assignment for the machines involved.
