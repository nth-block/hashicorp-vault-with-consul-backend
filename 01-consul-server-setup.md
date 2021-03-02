# Consul Server setup 

## Consul-1 server

> Repeat the steps in this chapter for all consul members

### Download consul binaries

```
export VER=1.9.3
wget https://releases.hashicorp.com/consul/${VER}/consul_${VER}_linux_amd64.zip
sudo apt-get install -y unzip
sudo unzip consul_${VER}_linux_amd64.zip
sudo chmod +x consul
sudo mv consul /usr/local/bin/
```

### Create the necessary users

```
sudo groupadd consul
sudo useradd -s /sbin/nologin -g consul consul
```

### Create the data_dir and config for the consul server

```
sudo mkdir -p /var/consul/data
sudo chown -R consul:consul /var/consul/data
sudo mkdir -p /usr/local/etc/consul/

```

### Consul-1 server config 
 
This file must be saved as `/usr/local/etc/consul/consul-1.json`

```
{
  "server": true,
  "node_name": "consul-1",
  "datacenter": "dc1",
  "data_dir": "/var/consul/data",
  "bind_addr": "0.0.0.0",
  "client_addr": "0.0.0.0",
  "advertise_addr": "10.0.0.4",
  "bootstrap_expect": 2,
  "retry_join": ["10.0.0.4", "10.0.0.5"],
  "ui": true,
  "log_level": "DEBUG",
  "enable_syslog": true,
  "acl_enforce_version_8": false
}
```

### Consul-1 server systemctl script configuration

This file must be saved as `/etc/systemd/system/consul.service`

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
Description=Consul server agent
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
    -config-file=/usr/local/etc/consul/consul-1.json \
    -pid-file=/var/run/consul/consul.pid
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
KillSignal=SIGTERM
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```

### Starting the consul service

Only for the first time run `sudo systemctl daemon-reload`

`sudo systemctl start consul`

### Check the status of the consul service

Crucially, the **Active** state must be *active (running)*

```
sudo systemctl status consul

● consul.service - Consul server agent
   Loaded: loaded (/etc/systemd/system/consul.service; disabled; vendor preset: enabled)
   Active: active (running) since Mon 2021-03-01 12:50:50 UTC; 33s ago
  Process: 3065 ExecStartPre=/bin/chown -R consul:consul /var/run/consul (code=exited, status=0/SUCCESS)
  Process: 3060 ExecStartPre=/bin/mkdir -p /var/run/consul (code=exited, status=0/SUCCESS)
 Main PID: 3074 (consul)
    Tasks: 10 (limit: 4680)
   CGroup: /system.slice/consul.service
           └─3074 /usr/local/bin/consul agent -config-file=/usr/local/etc/consul/consul-1.json -pid-file=/var/run/consul/consul.pid

Mar 01 12:50:50 consul-1 consul[3074]: 2021-03-01T12:50:50.926Z [DEBUG] agent.server.memberlist.lan: memberlist: Failed to join 10.0.0.5: dial tcp 10.0.0.5:83
Mar 01 12:50:50 consul-1 consul[3074]: 2021-03-01T12:50:50.926Z [INFO]  agent: (LAN) joined: number_of_nodes=1
Mar 01 12:50:50 consul-1 consul[3074]: 2021-03-01T12:50:50.926Z [DEBUG] agent: systemd notify failed: error="No socket"
Mar 01 12:50:50 consul-1 consul[3074]: 2021-03-01T12:50:50.926Z [INFO]  agent: Join cluster completed. Synced with initial agents: cluster=LAN num_agents=1
Mar 01 12:50:55 consul-1 consul[3074]:     2021-03-01T12:50:55.924Z [WARN]  agent.server.raft: no known peers, aborting election
Mar 01 12:50:55 consul-1 consul[3074]: 2021-03-01T12:50:55.924Z [WARN]  agent.server.raft: no known peers, aborting election
Mar 01 12:50:58 consul-1 consul[3074]:     2021-03-01T12:50:58.168Z [ERROR] agent.anti_entropy: failed to sync remote state: error="No cluster leader"
Mar 01 12:50:58 consul-1 consul[3074]: 2021-03-01T12:50:58.168Z [ERROR] agent.anti_entropy: failed to sync remote state: error="No cluster leader"
Mar 01 12:51:18 consul-1 consul[3074]:     2021-03-01T12:51:18.071Z [ERROR] agent: Coordinate update error: error="No cluster leader"
Mar 01 12:51:18 consul-1 consul[3074]: 2021-03-01T12:51:18.071Z [ERROR] agent: Coordinate update error: error="No cluster leader"

```

## Common tests

### Consul members

```
consul members

Node      Address        Status  Type    Build  Protocol  DC   Segment
consul-1  10.0.0.4:8301  alive   server  1.9.3  2         dc1  <all>
consul-2  10.0.0.5:8301  alive   server  1.9.3  2         dc1  <all>
```

### Consul peers

Crucially, one must be the leader

```
consul operator raft list-peers

Node      ID                                    Address        State     Voter  RaftProtocol
consul-1  9d8f9713-4eb6-464b-8dad-cf8e6d899243  10.0.0.4:8300  leader    true   3
consul-2  ac18c5b4-6bd9-58cc-ad7d-faecd4aac559  10.0.0.5:8300  follower  true   3
```

## Making the DNS settings

`sudo vim.tiny /etc/hosts`

```
# Added for the Hashicorp testing

10.0.0.4 consul-1.purplepandas.org consul-1
10.0.0.5 consul-2.purplepandas.org consul-2
10.0.0.6 vault-1.purplepandas.org vault-1
```



# References

https://computingforgeeks.com/how-to-install-consul-cluster-18-04-lts/
https://learn.hashicorp.com/tutorials/vault/ha-with-consul