# Failover Tests

Run against the live 3-node cluster (`pg-ha-lab`) through HAProxy at `192.168.56.20:5432`.
Every result below is from an actual run.

All commands below run from the `ansible/` directory (so `ansible.cfg`'s inventory
resolves) unless noted otherwise. Everywhere a node name is needed, run
`patronictl list` first and substitute whichever node is actually Leader/Replica at
that moment. Roles shift as you go through these tests, don't assume pg-01 is always the leader.

## Setup — run once

Create a non-superuser test role (apps shouldn't connect as `postgres`):
```bash
cd ansible/
ansible pg-01 -m shell -a "PGPASSWORD='{{ patroni_superuser_password }}' psql -h 192.168.56.20 -U postgres -d postgres -c \"CREATE ROLE app_user LOGIN PASSWORD 'test_password' CREATEDB;\""
```
`{{ patroni_superuser_password }}` is templated by Ansible from
`inventory/group_vars/postgres/vault.yml` (gitignored, copy `vault.yml.example` to
`vault.yml` and fill in real values before running anything in this repo).

PostgreSQL 15+ revokes `CREATE` on the `public` schema from new roles by default. Grant it once:
```bash
ansible pg-01 -m shell -a "PGPASSWORD='{{ patroni_superuser_password }}' psql -h 192.168.56.20 -U postgres -d postgres -c 'GRANT CREATE ON SCHEMA public TO app_user;'"
```

Create the table these tests write to:
```bash
ansible pg-01 -m shell -a "PGPASSWORD=test_password psql -h 192.168.56.20 -U app_user -d postgres -c 'CREATE TABLE IF NOT EXISTS ha_test (id serial primary key, note text, created_at timestamptz default now());'"
```

**Note on killing nodes:** every test below that "fails" a node uses
`VBoxManage controlvm <name> poweroff` (hard power cut), never `vagrant halt` — a
graceful ACPI shutdown gives Patroni time to demote cleanly, which isn't a real
failure and won't exercise the automatic-failover path. VM names are
`patroni-lab-<inventory-name>`, e.g. `patroni-lab-pg-02`.

## Test 1 — cluster status

```bash
ansible pg-01 -b -m shell -a "/opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml list"
```
**Expect:** exactly one `Leader` + two `Replica`, both `streaming`, lag `0`.

## Test 2 — write path through HAProxy

```bash
ansible pg-01 -m shell -a "PGPASSWORD=test_password psql -h 192.168.56.20 -U app_user -d postgres -c \"INSERT INTO ha_test (note) VALUES ('test2-write-path') RETURNING *;\""
```
**Expect:** `INSERT 0 1`. Client only ever talks to `192.168.56.20`, not a 
specific node and it lands on whichever one is currently primary.

Optional: confirm the row replicated, reading directly from each replica (bypassing
HAProxy, which only routes to the primary):
```bash
ansible pg-02 -m shell -a "PGPASSWORD=test_password psql -h 192.168.56.12 -U app_user -d postgres -c 'SELECT * FROM ha_test;'"
ansible pg-03 -m shell -a "PGPASSWORD=test_password psql -h 192.168.56.13 -U app_user -d postgres -c 'SELECT * FROM ha_test;'"
```

## Test 3 — replica failure

Check current roles, then power off a **replica** (not the leader):
```bash
ansible pg-01 -b -m shell -a "/opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml list"
VBoxManage controlvm patroni-lab-pg-02 poweroff   # substitute the actual replica's name
```

Confirm the leader is unaffected and writes keep working:
```bash
ansible pg-01 -m shell -a "PGPASSWORD=test_password psql -h 192.168.56.20 -U app_user -d postgres -c \"INSERT INTO ha_test (note) VALUES ('test3-replica-failure') RETURNING *;\""
```
**Expect:** insert succeeds normally. (`patronictl list` may still show the dead 
replica as `streaming` for up to ~30s, it's reading its last-known status from etcd, 
which only goes stale once that node's lease expires; this isn't a bug.)

Bring it back and confirm it rejoins:
```bash
cd ../vagrant && vagrant up pg-02 && cd ../ansible
ansible pg-01 -b -m shell -a "/opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml list"
```
**Expect:** back to one Leader + two streaming Replicas, lag `0`, including the row
written while it was down.

## Test 4 — controlled switchover

```bash
ansible pg-01 -b -m shell -a "/opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml list"
ansible pg-01 -b -m shell -a "/opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml switchover --leader <current-leader> --candidate <a-replica> --force"
ansible pg-01 -b -m shell -a "/opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml list"
```
Replace <current-leader> and <a-replica> with actual VM node names.

**Expect:** the named candidate is now `Leader`; the old leader is now a streaming
`Replica`.

Confirm the client endpoint didn't change:
```bash
ansible pg-01 -m shell -a "PGPASSWORD=test_password psql -h 192.168.56.20 -U app_user -d postgres -c \"INSERT INTO ha_test (note) VALUES ('test4-switchover') RETURNING *, inet_server_addr();\""
```
**Expect:** `inet_server_addr()` shows the new leader's IP. Give it ~5–10s first.
HAProxy's default health-check interval needs a couple of cycles to flip.

## Test 5 — leader loss (automatic failover)

Power off whoever is currently leader:
```bash
ansible pg-01 -b -m shell -a "/opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml list"
VBoxManage controlvm patroni-lab-pg-03 poweroff   # substitute the actual leader's name
```

Wait ~30–40s (the leader lock has to actually expire. `ttl: 30`), then:
```bash
ansible pg-01 -b -m shell -a "/opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml list"
```
**Expect:** one of the two surviving nodes auto-promoted to `Leader`. Nobody ran `switchover`.

## Test 6 — reconnect

```bash
ansible pg-01 -m shell -a "PGPASSWORD=test_password psql -h 192.168.56.20 -U app_user -d postgres -c \"INSERT INTO ha_test (note) VALUES ('test6-reconnect') RETURNING *, inet_server_addr();\""
```
**Expect:** succeeds, `inet_server_addr()` shows the newly-promoted leader. Same address as always (`192.168.56.20`).

## Test 7 — old leader rejoin

Bring the node killed in test 5 back:
```bash
cd ../vagrant && vagrant up pg-03 && cd ../ansible   # substitute the actual node
ansible pg-01 -b -m shell -a "/opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml list"
```
**Expect:** it rejoins as a **Replica** of the new leader not a second primary.

See how it resynced:
```bash
ansible pg-03 -b -m shell -a "journalctl -u patroni --no-pager | grep -iE 'rewind|timeline|following' | tail -15"
```
If it was killed before accepting any writes the new leader doesn't have, it just
fetches the new timeline's history file and resumes streaming, so no rewind would be needed.
`pg_rewind` (`use_pg_rewind: true` + `data-checksums` + `wal_log_hints`) exists for
the harder case: an old leader that *did* accept writes its replacement never got.

## Durability questions

**Why can Patroni's async mode lose committed transactions?**
Async is the default: the primary acknowledges `COMMIT` the moment it flushes WAL
locally, before any replica has necessarily received or applied it. If the primary
dies right after that acknowledgment and before a replica catches up, that
transaction only ever existed on the dead primary's disk. However the client was told it
was committed, the cluster that survives doesn't have it.

**What does `maximum_lag_on_failover` limit, and what does it NOT guarantee?**
It's an eligibility filter. A replica whose measured WAL lag exceeds the byte threshold is 
excluded as a promotion candidate. It does not guarantee zero data loss: it only measures 
lag at the moment it's checked, and a candidate under the threshold can still be missing 
the very last committed transactions.

**What changes when `synchronous_mode` is enabled?**
With synchronous_mode, Patroni tries to make sure that a transaction acknowledged as 
committed also exists on a synchronous standby. This greatly reduces the risk of losing 
acknowledged transactions when the primary fails. The downside is that writes can become 
slower or stop if a suitable synchronous replica is unavailable.

**Why doesn't lower failover time automatically mean a better design?**
Faster `ttl`/`loop_wait` makes Patroni declare a leader dead sooner but also raises the 
odds of a false positive (a leader that's just slow, not actually dead) triggering an 
unnecessary promotion. Fast recovery from a real failure and stability against false alarms 
pull in opposite directions.

**What role do `ttl`, `loop_wait`, and `retry_timeout` play?**

`ttl`: how long the leader's lock remains valid before it expires.

`loop_wait`: how often Patroni checks things and performs its loop.

`retry_timeout`: how long Patroni is willing to keep retrying a failed DCS/PostgreSQL operation.

The rule `loop_wait + 2*retry_timeout <= ttl` stops a merely-slow-but-alive leader from 
losing its lock before it even gets a retry.

**Why is watchdog/fencing still a production consideration?**
Patroni's demote-before-expiry behavior is cooperative and software-level. It works
because the process is still alive and able to act. If the OS or VM freezes instead
of cleanly stopping, that demote step never runs, and once the lock expires
elsewhere you can end up with two nodes both believing they're primary. A watchdog
device forces a hard reset if the leader can't prove it's still in control which is a
guarantee that software alone can't provide.

The key concept here is split-brain prevention.

