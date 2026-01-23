# README.md – PostgreSQL HA on Ubuntu (Percona PPG 15 + Patroni + Etcd + HAProxy)

This document describes how to deploy a **high availability PostgreSQL cluster** using:

* **Percona Distribution for PostgreSQL (PPG 15)**
* **Patroni** (HA / leader election)
* **Etcd** (DCS)
* **HAProxy** (routing write/read traffic)

---

## 0) Topology (Example)

### Nodes

* **node1 / server-ps-1**: `192.168.41.120`
* **node2 / server-ps-2**: `192.168.41.121`

> You can extend to node3 with the same steps.

### Ports

* Etcd client: `2379`
* Etcd peer: `2380`
* Patroni REST API: `8008`
* PostgreSQL: `5432`
* HAProxy:

  * Primary (write): `5000`
  * Standby (read): `5001`
  * Stats UI: `7000`

---

## 1) Install Percona PPG 15 + Patroni + Etcd packages (All DB nodes)

### 1.1 Install Percona repository

```bash
wget https://repo.percona.com/apt/percona-release_latest.$(lsb_release -sc)_all.deb
sudo dpkg -i percona-release_latest.$(lsb_release -sc)_all.deb
sudo apt update
```

Enable PPG 15 repo:

```bash
sudo percona-release setup ppg-15
```

### 1.2 Install PostgreSQL server

```bash
sudo apt install -y percona-ppg-server-15
```

### 1.3 Install supporting packages

```bash
sudo apt install -y python3-pip python3-dev binutils
sudo apt install -y percona-patroni
sudo apt install -y percona-pgbackrest
sudo apt install -y etcd etcd-server etcd-client
```

### 1.4 Stop + disable services (will be managed by Patroni later)

```bash
sudo systemctl stop etcd patroni postgresql
sudo systemctl disable etcd patroni postgresql
```

### 1.5 Remove old PG data dir (clean install)

```bash
sudo rm -rf /var/lib/postgresql/15/main
```

---

## 2) Setup Etcd Cluster

### 2.1 Configure Etcd on node1 (server-ps-1)

Edit config:

```bash
sudo vi /etc/etcd/etcd.conf.yaml
```

Example:

```yaml
name: 'server-ps-1'

data-dir: /var/lib/etcd

initial-cluster-token: postgresql_cluster_ducanh
initial-cluster-state: new
initial-cluster: server-ps-1=http://192.168.41.120:2380

initial-advertise-peer-urls: http://192.168.41.120:2380
listen-peer-urls: http://0.0.0.0:2380

advertise-client-urls: http://192.168.41.120:2379
listen-client-urls: http://0.0.0.0:2379
```

Start etcd:

```bash
sudo systemctl start etcd
sudo systemctl enable etcd
```

Verify members:

```bash
etcdctl --endpoints=http://192.168.41.120:2379 member list
```

---

### 2.2 Add node2 to Etcd cluster (run on node1)

```bash
etcdctl --endpoints=http://192.168.41.120:2379 member add server-ps-2 --peer-urls=http://192.168.41.121:2380
```

---

### 2.3 Configure Etcd on node2 (server-ps-2)

Edit config:

```bash
sudo vi /etc/etcd/etcd.conf.yaml
```

Example:

```yaml
name: 'server-ps-2'

data-dir: /var/lib/etcd

initial-cluster-token: postgresql_cluster_ducanh
initial-cluster-state: existing
initial-cluster: server-ps-1=http://192.168.41.120:2380,server-ps-2=http://192.168.41.121:2380

initial-advertise-peer-urls: http://192.168.41.121:2380
listen-peer-urls: http://0.0.0.0:2380

advertise-client-urls: http://192.168.41.121:2379
listen-client-urls: http://0.0.0.0:2379
```

Start etcd:

```bash
sudo systemctl start etcd
sudo systemctl enable etcd
```

Verify etcd cluster:

```bash
etcdctl --endpoints=http://192.168.41.120:2379 member list
```

---

## 3) Setup Patroni

### 3.1 Export environment variables (All DB nodes)

Run:

```bash
export NODE_NAME=$(hostname -f)
export NODE_IP=$(getent hosts $(hostname -f) | awk '{ print $1 }' | grep -v '127.0.1.1')
export DATA_DIR="/var/lib/postgresql/15/main"
export PG_BIN_DIR="/usr/lib/postgresql/15/bin"
export NAMESPACE="percona_lab"
export SCOPE="cluster_1"
```

---

### 3.2 Create Patroni config (All DB nodes)

Create config:

```bash
sudo tee /etc/patroni/patroni.yml >/dev/null <<EOF
namespace: ${NAMESPACE}
scope: ${SCOPE}
name: ${NODE_NAME}

restapi:
  listen: 0.0.0.0:8008
  connect_address: ${NODE_IP}:8008

etcd3:
  host: ${NODE_IP}:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576

    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        wal_keep_segments: 10
        max_wal_senders: 5
        max_replication_slots: 10
        wal_log_hints: "on"
        logging_collector: 'on'
        max_wal_size: '10GB'
        archive_mode: "on"
        archive_timeout: 600s
        archive_command: "cp -f %p /home/postgres/archived/%f"

  pg_hba:
  - host replication replicator 127.0.0.1/32 trust
  - host replication replicator 0.0.0.0/0 md5
  - host all all 0.0.0.0/0 md5
  - host all all ::0/0 md5

  recovery_conf:
    restore_command: cp /home/postgres/archived/%f %p

  initdb:
  - encoding: UTF8
  - data-checksums

postgresql:
  cluster_name: ${SCOPE}
  listen: 0.0.0.0:5432
  connect_address: ${NODE_IP}:5432
  data_dir: ${DATA_DIR}
  bin_dir: ${PG_BIN_DIR}
  pgpass: /tmp/pgpass0

  authentication:
    replication:
      username: replicator
      password: replPasswd
    superuser:
      username: postgres
      password: qaz123

  parameters:
    unix_socket_directories: "/var/run/postgresql/"

  create_replica_methods:
  - basebackup
  basebackup:
    checkpoint: 'fast'

watchdog:
  mode: required
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
EOF
```

---

### 3.3 Create `/var/run/postgresql` directory (All DB nodes)

```bash
sudo mkdir -p /var/run/postgresql
sudo chown postgres:postgres /var/run/postgresql
sudo chmod 2775 /var/run/postgresql
```

---

### 3.4 Enable + start Patroni (All DB nodes)

```bash
sudo systemctl enable patroni
sudo systemctl start patroni
```

---

## 4) Fix after EC2 reboot (important)

Sometimes `/var/run/postgresql` disappears (tmpfs). Run:

```bash
sudo mkdir -p /var/run/postgresql
sudo chown postgres:postgres /var/run/postgresql
sudo chmod 2775 /var/run/postgresql
```

Restart Patroni:

```bash
sudo systemctl reset-failed patroni
sudo systemctl restart patroni
sudo systemctl status patroni --no-pager -l
```

### 4.1 Auto-create folder using tmpfiles.d (recommended)

```bash
sudo tee /etc/tmpfiles.d/postgresql.conf >/dev/null <<'EOF'
d /var/run/postgresql 2775 postgres postgres -
EOF
```

Apply:

```bash
sudo systemd-tmpfiles --create
```

---

## 5) Fix replication stuck in "stopped" (pg_hba rules)

On **leader node**, add allowed replication hosts:

```bash
sudo tee -a /var/lib/postgresql/15/main/pg_hba.conf >/dev/null <<'EOF'
host replication replicator 192.168.41.121/32 md5
host replication replicator 192.168.41.122/32 md5
EOF
```

Reload config:

```bash
sudo -u postgres psql -c "select pg_reload_conf();"
```

Verify:

```bash
sudo -u postgres egrep -n "replication|192.168" /var/lib/postgresql/15/main/pg_hba.conf
```

---

## 6) Install + Configure HAProxy

### 6.1 Install HAProxy

```bash
sudo apt-get update
sudo apt install -y haproxy
```

Backup config:

```bash
sudo mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.org
sudo vi /etc/haproxy/haproxy.cfg
```

Example config:

```cfg
global
    maxconn 100

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /stats
    stats auth percona:myS3cr3tpass

listen primary
    bind *:5000
    option httpchk /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server node1 node1:5432 maxconn 100 check port 8008
    server node2 node2:5432 maxconn 100 check port 8008
    server node3 node3:5432 maxconn 100 check port 8008

listen standbys
    balance roundrobin
    bind *:5001
    option httpchk /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server node1 node1:5432 maxconn 100 check port 8008
    server node2 node2:5432 maxconn 100 check port 8008
    server node3 node3:5432 maxconn 100 check port 8008
```

Restart HAProxy:

```bash
sudo systemctl restart haproxy
sudo systemctl status haproxy --no-pager -l
```

---

## 7) Tighten security (allow only HAProxy node to connect)

If HAProxy runs on node1 (`192.168.41.120`), allow only this IP:

```bash
sudo tee -a /var/lib/postgresql/15/main/pg_hba.conf >/dev/null <<'EOF'
host all postgres 192.168.41.120/32 md5
EOF
```

Reload:

```bash
sudo -u postgres psql -c "select pg_reload_conf();"
```

---

## 8) Verify cluster

### Patroni status

```bash
sudo patronictl -c /etc/patroni/patroni.yml list
```

### Test HAProxy ports

Write (primary):

```bash
psql -h <haproxy_ip> -p 5000 -U postgres -d postgres
```

Read (standby):

```bash
psql -h <haproxy_ip> -p 5001 -U postgres -d postgres
```

---

DONE ✅

* Etcd configured
* Patroni manages PostgreSQL cluster
* HAProxy provides Primary/Standby routing
* Basic security hardening applied
