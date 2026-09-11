# PostgreSQL HA Cluster Lab — Patroni + etcd3 + HAProxy

PostgreSQL Lab, adapted to run **locally on Vagrant + VirtualBox**
instead of a cloud provider.

## What this builds

A 3-node PostgreSQL cluster where HA decisions are owned by Patroni, coordination
by a 3-member etcd cluster, and client routing by HAProxy. Runs PostgreSQL 17
today, with a planned major upgrade to 18.

| VM    | IP             | Role                            |
|-------|----------------|---------------------------------|
| pg-01 | 192.168.56.11  | PostgreSQL + Patroni + etcd     |
| pg-02 | 192.168.56.12  | PostgreSQL + Patroni + etcd     |
| pg-03 | 192.168.56.13  | PostgreSQL + Patroni + etcd     |
| lb-01 | 192.168.56.20  | HAProxy (stable write endpoint) |

Ports: 22 SSH · 5432 PostgreSQL · 8008 Patroni REST · 2379/2380 etcd

## Tooling

- VirtualBox
- Vagrant
- Ansible

**Vagrant** is used for provisioning VMs and **Ansible** for configuration. Leader election is done through Patroni + etcd at runtime.

## How to run
Boot up VMs:
```bash
cd vagrant
vagrant up             # create + boot the VMs (first run downloads the box)
vagrant status         # all VMs should say "running"
vagrant ssh pg-01      # log into a node
vagrant halt           # stop the VMs
vagrant destroy -f     # delete them
```

Verify the private network is up:

```bash
vagrant ssh pg-01 -c "ip -4 addr show eth1; hostname"
# expect 192.168.56.11 on eth1, hostname pg-01
```

Then configure the VMs with Ansible (run from `ansible/`):

```bash
cd ../ansible
ansible all -m ping                      # check connectivity
ansible-playbook playbooks/site.yml      # apply config; re-run should show changed=0
```

Verify:

```bash
ansible all -m shell -a "chronyc tracking | head -1; getent hosts pg-03"
# expect a chrony Reference ID and pg-03 -> 192.168.56.13 on every node
```

Verify etcd:

```bash
ETCD_EP="http://192.168.56.11:2379,http://192.168.56.12:2379,http://192.168.56.13:2379"
ansible pg-01 -m shell -a "/usr/local/bin/etcdctl --endpoints=$ETCD_EP member list -w table"
ansible pg-01 -m shell -a "/usr/local/bin/etcdctl --endpoints=$ETCD_EP endpoint status --cluster -w table"
ansible pg-01 -m shell -a "/usr/local/bin/etcdctl --endpoints=$ETCD_EP endpoint health --cluster"
ansible etcd -m shell -a "systemctl is-active etcd; systemctl is-enabled etcd"
# expect: 3 members "started", one "is leader: true", all endpoints "healthy", etcd "active"/"enabled" on pg-01..03
```

## Repo layout

```
vagrant/            
  Vagrantfile
ansible/
  inventory/
    hosts.ini
    group_vars/
  playbooks/site.yml
  roles/{common,etcd,postgres,patroni,haproxy}/
architecture.md
RUNBOOK.md
FAILOVER_TESTS.md
```
