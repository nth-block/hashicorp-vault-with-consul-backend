# Consul agents installation and configuration

These setups must be done in the vault.

> Vault VM needs consul as well as vault as service disovery (of backend data store) happens via the consul agents running in the vault VM

## Downloading the binaries

```
export VER=1.9.3
wget https://releases.hashicorp.com/consul/${VER}/consul_${VER}_linux_amd64.zip
sudo apt-get install -y unzip 
sudo unzip consul_${VER}_linux_amd64.zip
sudo mv consul /usr/local/bin/
```

## Setting up the consult agent

### Create the necessary users

```
sudo groupadd consul
sudo useradd -s /sbin/nologin -g consul consul
sudo groupadd vault
sudo useradd -s /sbin/nologin -g vault vault
```

### Create the data_dir and config for the consul server

```
sudo mkdir -p /var/consul/data
sudo chown -R consul:consul /var/consul/data
sudo mkdir -p /usr/local/etc/consul/
```

### Create the pid dir for the vault server

`sudo mkdir -p /var/run/vault/`

# Vault-1 Consul Agent Config

This file must be saved as `/usr/local/etc/consul/vault-1.json`

```
{
  "server": false,
  "datacenter": "dc1",
  "node_name": "consul-c1",
  "data_dir": "/var/consul/data",
  "bind_addr": "10.0.0.6",
  "client_addr": "127.0.0.1",
  "retry_join": ["10.0.0.4","10.0.0.5"],
  "log_level": "DEBUG",
  "enable_syslog": true,
  "acl_enforce_version_8": false
}
```
## Systemctl config script

This file must be saved as ``

```
### BEGIN INIT INFO
# Provides:          consul
# Required-Start:    $local_fs $remote_fs
# Required-Stop:     $local_fs $remote_fs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Consul agent
# Description:       Consul service discovery framework
### END INIT INFO

[Unit]
Description=Consul client agent
Requires=network-online.target
After=network-online.target

[Service]
User=consul
Group=consul
PIDFile=/var/run/consul/consul.pid
PermissionsStartOnly=true
ExecStartPre=-/bin/mkdir -p /var/run/consul
ExecStartPre=/bin/chown -R consul:consul /var/run/consul
ExecStart=/usr/local/bin/consul agent \
    -config-file=/usr/local/etc/consul/vault-1.json \
    -pid-file=/var/run/consul/consul.pid
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
KillSignal=SIGTERM
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```

## Starting the consul service

Only for the first time run `sudo systemctl daemon-reload`

```
sudo systemctl start consul
```

## Check the status of the consul service

Crucially, the **Active** state must be *active (running)*

```
sudo systemctl status consul

● consul.service - Consul client agent
   Loaded: loaded (/etc/systemd/system/consul.service; disabled; vendor preset: enabled)
   Active: active (running) since Mon 2021-03-01 16:39:45 UTC; 4s ago
  Process: 13677 ExecStartPre=/bin/chown -R consul:consul /var/run/consul (code=exited, status=0/SUCCESS)
  Process: 13676 ExecStartPre=/bin/mkdir -p /var/run/consul (code=exited, status=0/SUCCESS)
 Main PID: 13682 (consul)
    Tasks: 8 (limit: 4680)
   CGroup: /system.slice/consul.service
           └─13682 /usr/local/bin/consul agent -config-file=/usr/local/etc/consul/vault-1.json -pid-file=/var/run/consul/consul.pid

Mar 01 16:39:47 vault-1 consul[13682]:     2021-03-01T16:39:47.321Z [DEBUG] agent: Skipping remote check since it is managed automatically: check=serfHealth
Mar 01 16:39:47 vault-1 consul[13682]: 2021-03-01T16:39:47.321Z [DEBUG] agent: Skipping remote check since it is managed automatically: check=serfHealth
Mar 01 16:39:47 vault-1 consul[13682]:     2021-03-01T16:39:47.400Z [INFO]  agent: Synced node info
Mar 01 16:39:47 vault-1 consul[13682]: 2021-03-01T16:39:47.400Z [INFO]  agent: Synced node info
Mar 01 16:39:50 vault-1 consul[13682]:     2021-03-01T16:39:50.098Z [DEBUG] agent: Skipping remote check since it is managed automatically: check=serfHealth
Mar 01 16:39:50 vault-1 consul[13682]:     2021-03-01T16:39:50.098Z [DEBUG] agent: Node info in sync
Mar 01 16:39:50 vault-1 consul[13682]:     2021-03-01T16:39:50.098Z [DEBUG] agent: Node info in sync
Mar 01 16:39:50 vault-1 consul[13682]: 2021-03-01T16:39:50.098Z [DEBUG] agent: Skipping remote check since it is managed automatically: check=serfHealth
Mar 01 16:39:50 vault-1 consul[13682]: 2021-03-01T16:39:50.098Z [DEBUG] agent: Node info in sync
Mar 01 16:39:50 vault-1 consul[13682]: 2021-03-01T16:39:50.098Z [DEBUG] agent: Node info in sync
```

## Common tests

### View all the participants in the consul cluster

```
consul members

consul-1   10.0.0.4:8301  alive   server  1.9.3  2         dc1  <all>
consul-2   10.0.0.5:8301  alive   server  1.9.3  2         dc1  <all>
consul-c1  10.0.0.6:8301  alive   client  1.9.3  2         dc1  <default>
```

# Vault installation and configuration

## Downloading the vault binaries

```
export VER=1.6.3
wget https://releases.hashicorp.com/vault/${VER}/vault_${VER}_linux_amd64.zip
unzip vault_${VER}_linux_amd64.zip
sudo chmod +x vault
sudo mv vault /usr/local/bin/
```

## Setup the config paths

```
sudo mkdir -p /usr/local/etc/vault/
```

## Vault server configuration

This file must be saved as `/usr/local/etc/vault/vault-config.hcl`

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

api_addr =  "http://10.0.0.6:8200"
cluster_addr = "https://10.0.0.6:8201"
```

## Vault server systemctl config script

This file must be saved as `/etc/systemd/system/vault.service`

```
### BEGIN INIT INFO
# Provides:          vault
# Required-Start:    $local_fs $remote_fs
# Required-Stop:     $local_fs $remote_fs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Vault server
# Description:       Vault secret management tool
### END INIT INFO

[Unit]
Description=Vault secret management tool
Requires=network-online.target
After=network-online.target

[Service]
User=vault
Group=vault
PIDFile=/var/run/vault/vault.pid
ExecStart=/usr/local/bin/vault server -config=/usr/local/etc/vault/vault-config.hcl -log-level=debug
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
KillSignal=SIGTERM
Restart=on-failure
RestartSec=42s
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
```

## Starting the vault server service

Only for the first time run `sudo systemctl daemon-reload`

`sudo systemctl start vault`

## Check the status of the consul service

Crucially, the **Active** state must be *active (running)*

```
sudo systemctl status vault

● vault.service - Vault secret management tool
   Loaded: loaded (/etc/systemd/system/vault.service; disabled; vendor preset: enabled)
   Active: active (running) since Mon 2021-03-01 17:29:50 UTC; 2min 41s ago
 Main PID: 15280 (vault)
    Tasks: 7 (limit: 4680)
   CGroup: /system.slice/vault.service
           └─15280 /usr/local/bin/vault server -config=/usr/local/etc/vault/vault-config.hcl -log-level=debug

Mar 01 17:29:50 vault-1 vault[15280]: 2021-03-01T17:29:50.104Z [DEBUG] service_registration.consul: config disable_registration set: disable_registration=fals
Mar 01 17:29:50 vault-1 vault[15280]: 2021-03-01T17:29:50.104Z [DEBUG] service_registration.consul: config service set: service=vault
Mar 01 17:29:50 vault-1 vault[15280]: 2021-03-01T17:29:50.104Z [DEBUG] service_registration.consul: config service_tags set: service_tags=
Mar 01 17:29:50 vault-1 vault[15280]: 2021-03-01T17:29:50.104Z [DEBUG] service_registration.consul: config service_address set: service_address=<nil>
Mar 01 17:29:50 vault-1 vault[15280]: 2021-03-01T17:29:50.104Z [DEBUG] service_registration.consul: config address set: address=127.0.0.1:8500
Mar 01 17:29:50 vault-1 vault[15280]: 2021-03-01T17:29:50.104Z [DEBUG] core: set config: sanitized config={"api_addr":"http://10.0.0.6:8200","cache_size":0,"c
Mar 01 17:29:50 vault-1 vault[15280]: 2021-03-01T17:29:50.104Z [DEBUG] storage.cache: creating LRU cache: size=0
Mar 01 17:29:50 vault-1 vault[15280]: 2021-03-01T17:29:50.148Z [DEBUG] cluster listener addresses synthesized: cluster_addresses=[10.0.0.6:8201]
Mar 01 17:30:16 vault-1 vault[15280]: 2021-03-01T17:30:16.559Z [INFO]  core: security barrier not initialized
Mar 01 17:30:16 vault-1 vault[15280]: 2021-03-01T17:30:16.563Z [INFO]  core: seal configuration missing, not initialized
```

# Common tests

## Set the VAULT_ADDR environment variable

```
export VAULT_ADDR=http://127.0.0.1:8200
```

## Vault status

`vault status`

```
Key                Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       0
Threshold          0
Unseal Progress    0/0
Unseal Nonce       n/a
Version            1.6.3
Storage Type       consul
HA Enabled         true
```

## Initialize the vault

```
vault operator init

Unseal Key 1: ********************************************
Unseal Key 2: ********************************************
Unseal Key 3: ********************************************
Unseal Key 4: ********************************************
Unseal Key 5: ********************************************

Initial Root Token: s.************************

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 3 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```

> Save these above unseal keys and root token very securely.

## Making the DNS settings

`sudo vim.tiny /etc/hosts`

```
# Added for the Hashicorp testing

10.0.0.4 consul-1.purplepandas.org consul-1
10.0.0.5 consul-2.purplepandas.org consul-2
10.0.0.6 vault-1.purplepandas.org vault-1
```

# References

https://learn.hashicorp.com/tutorials/vault/ha-with-consul

