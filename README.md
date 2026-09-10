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

```bash
cd vagrant
vagrant up             # create + boot the 4 VMs (first run downloads the box)
vagrant status         # all 4 should say "running"
vagrant ssh pg-01      # log into a node
vagrant halt           # stop the VMs
vagrant destroy -f     # delete them
```

Verify the private network is up:

```bash
vagrant ssh pg-01 -c "ip -4 addr show eth1; hostname"
# expect 192.168.56.11 on eth1, hostname pg-01
```

## Repo layout

```
vagrant/            
  Vagrantfile
ansible/
  inventory/
  group_vars/
  playbooks/site.yml
  roles/{common,etcd,postgres,patroni,haproxy}/
architecture.md
RUNBOOK.md
FAILOVER_TESTS.md
```
