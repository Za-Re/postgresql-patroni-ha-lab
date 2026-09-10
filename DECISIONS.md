# Decisions

Short log of choices and why. Newest on top.

## Vagrant + VirtualBox instead of Terraform + cloud
No aws free tier is available. Vagrant gives all VMs from one file and `vagrant destroy` cleans up.
Provisioning happens through Vagrant and configuration via Ansible — the lab's split is kept.

## Start on PostgreSQL 17, upgrade to 18 later
Install one major behind so we can run and document a real major-version upgrade
at the end.

## etcd co-located on pg-01/02/03
Compact for a laptop. Real cost: a pg node dying also takes an etcd member. It's
acceptable here, not in production.

## Private network 192.168.56.0/24
pg-01 .11, pg-02 .12, pg-03 .13, lb-01 .20. This is VirtualBox's default-allowed
host-only range, so no /etc/vbox/networks.conf edit is needed. There's no clash with the host LAN or other interfaces.

## Ansible runs from the host
The host should have Ansible. VMs are reachable on the private network.
