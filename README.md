# LinuxONE multi-LPAR Provisioning — Ansible

End-to-end automation for provisioning **one or many** LinuxONE partitions
(DPM mode) in a single playbook run. Each partition gets its own NICs
(by PCHID), FCP storage group, boot + data volumes, SG attachment, and
WWPN readback. At the end you get a single consolidated report file
listing every partition's WWPNs — perfect for one SAN-team ticket.

Built on IBM's official `ibm.ibm_zhmc` Ansible collection which talks to
the HMC REST API. No SSH to the CPC needed.

## Layout

```
lpar/
├── requirements.yml           # ibm.ibm_zhmc collection pin
├── inventory.yml              # localhost — HMC API runs from control node
├── provision-lpars.yml        # MAIN — loop over partitions, build report
├── decommission-lpars.yml     # tear down all (or subset)
├── tasks/
│   ├── create-partition.yml   # per-partition logic (9 phases)
│   └── delete-partition.yml   # per-partition teardown
├── vars/
│   ├── connection.yml         # HMC creds (vault-protected)
│   └── partitions.yml         # ★ LIST of partitions ★
├── README.md                  # this file
└── provisioning-report.txt    # ← generated; consolidated WWPN list
```

## What the multi-LPAR workflow does, step by step

```
PRE-FLIGHT
   ├─ Confirm CPC exists and is in DPM mode
   ├─ List all CPC adapters ONCE  → cache PCHID lookup
   └─ Pre-validate every PCHID across every partition (early abort)

LOOP over vars/partitions.yml:
   for each entry `p`:
       1. Create partition (stopped)             zhmc_partition
       2. Validate PCHIDs                         set_fact + assert
       3. Resolve adapter port URIs               set_fact
       4. Create NICs                             zhmc_nic (loop)
       5. Create FCP storage group                zhmc_storage_group
       6. Create boot + data volumes              zhmc_storage_volume (loop)
       7. Attach SG → partition                   zhmc_storage_group_attachment
       8. Set boot device                         zhmc_partition
       9. Read back WWPNs (capture into facts)    zhmc_partition state=facts
      10. Optional autostart                      zhmc_partition state=active
      11. Pause inter_partition_pause seconds

POST
   ├─ Print consolidated table of all partitions, their WWPNs, status
   ├─ Write provisioning-report.txt with one ticket-ready SAN ZONING block
   └─ Hard-fail if any partition failed AND fail_fast was off
```

## The `partitions` list (the file you'll edit most)

`vars/partitions.yml` is the single source of truth. Add / remove / edit
entries; the playbook loops over them. Each entry is fully self-contained:

```yaml
fail_fast: true                    # abort batch on first failure?
inter_partition_pause: 5           # seconds between partitions

partitions:
  - name: linuxone-app01
    description: "Production app workload"
    type: linux
    processor_mode: shared
    ifl_processors: 4
    initial_memory: 16384
    maximum_memory: 32768
    autostart: false

    network_adapters:
      - { name: nic-prod-data, pchid: "0140", port: 0, vlan_id: 100, description: "Prod — VLAN 100" }
      - { name: nic-mgmt,      pchid: "0141", port: 0, vlan_id: 200, description: "Mgmt — VLAN 200" }

    storage:
      group_name: "sg-app01"
      connectivity: 2              # → 2 WWPNs (multipath)
      boot_volume:
        name: "vol-boot-app01"
        size: 50
      data_volumes:
        - { name: "vol-data-app01-01", size: 200, description: "/var/lib/app" }

  - name: linuxone-db01
    type: linux
    processor_mode: dedicated      # dedicated IFLs for predictable lat
    ifl_processors: 8
    initial_memory: 65536
    autostart: false

    network_adapters:
      - { name: nic-prod-data,   pchid: "0140", port: 0, vlan_id: 100, description: "App data" }
      - { name: nic-replication, pchid: "0142", port: 0, vlan_id: 500, description: "DB repl" }

    storage:
      group_name: "sg-db01"
      connectivity: 4              # 4 paths for higher throughput
      boot_volume: { name: "vol-boot-db01", size: 80 }
      data_volumes:
        - { name: "vol-db01-data",   size: 1000, description: "/var/lib/pgsql/data" }
        - { name: "vol-db01-wal",    size: 200,  description: "WAL" }
```

Every partition can have a different shape: different IFLs, different
PCHID list, different storage layout. Loop processes them in declaration
order.

## One-time setup

```bash
pip install zhmcclient
ansible-galaxy collection install -r requirements.yml

# Encrypt the HMC password
ansible-vault create vars/vault.yml
# add inside the editor:
#   vault_hmc_password: <the-password>
```

## Run

```bash
cd outputs/lpar/

# Dry run — shows every change without touching the HMC
ansible-playbook -i inventory.yml provision-lpars.yml --check --diff --ask-vault-pass

# For real (default fail_fast=true: stop on first partition failure)
ansible-playbook -i inventory.yml provision-lpars.yml --ask-vault-pass

# Best-effort batch (continue even if one partition fails — useful for
# bulk dev environment refreshes where you'd rather see what worked)
ansible-playbook -i inventory.yml provision-lpars.yml --ask-vault-pass \
    -e fail_fast=false

# Slower, more polite to a busy HMC (10s between each partition)
ansible-playbook -i inventory.yml provision-lpars.yml --ask-vault-pass \
    -e inter_partition_pause=10
```

## What you get back

**1. Console summary at end of run:**

```
================================================================
MULTI-LPAR PROVISIONING SUMMARY  (4 partition(s))
CPC: Z16-A01    HMC: hmc.fmr.com
================================================================
OK  —  linuxone-app01
    IFLs / Memory  : 4 IFL  /  16384 MB
    Storage group  : sg-app01  (2 paths)
    Boot volume    : vol-boot-app01
    Data volumes   : vol-data-app01-01, vol-data-app01-02, vol-data-app01-03
    NICs           : nic-prod-data, nic-mgmt, nic-backup, nic-storage
    PCHIDs         : 0140, 0141, 0150, 0151
    WWPNs          : C05076D9C0001234, C05076D9C0005678
OK  —  linuxone-app02
    ...
OK  —  linuxone-db01
    IFLs / Memory  : 8 IFL  /  65536 MB
    Storage group  : sg-db01  (4 paths)
    WWPNs          : C05076D9C000ABCD, C05076D9C000ABCE, C05076D9C000ABCF, C05076D9C000ABD0
OK  —  linuxone-dev01
    ...
================================================================
```

**2. `provisioning-report.txt` in the playbook directory — the one file
your SAN team needs:**

```
[linuxone-app01]
status         = OK
ifls           = 4
storage_group  = sg-app01
connectivity   = 2
nics           = nic-prod-data, nic-mgmt, nic-backup, nic-storage
pchids         = 0140, 0141, 0150, 0151
wwpn_1         = C05076D9C0001234
wwpn_2         = C05076D9C0005678
...

# ============================================================
# SAN ZONING — give this whole list to your storage admin:
# ============================================================
# linuxone-app01 (sg-app01):
    C05076D9C0001234
    C05076D9C0005678
# linuxone-app02 (sg-app02):
    C05076D9C0009ABC
    C05076D9C0009DEF
# linuxone-db01 (sg-db01):
    C05076D9C000ABCD
    C05076D9C000ABCE
    C05076D9C000ABCF
    C05076D9C000ABD0
# linuxone-dev01 (sg-dev01):
    C05076D9C000F123
```

Drop this file (or just the SAN ZONING section at the bottom) into your
storage team's ticket and the whole batch can be zoned in one operation.

## Decommission

Tear down everything in `vars/partitions.yml`:

```bash
ansible-playbook -i inventory.yml decommission-lpars.yml --ask-vault-pass
```

Or just a subset by name (CSV):

```bash
ansible-playbook -i inventory.yml decommission-lpars.yml --ask-vault-pass \
    -e only=linuxone-dev01,linuxone-app02
```

There's an interactive confirmation pause that lists exactly what's
about to be destroyed.

## Behavior under failure

| `fail_fast` | What happens when partition #2 of 4 fails |
|---|---|
| `true` (default) | Batch aborts immediately. Partition #1 keeps its (already-completed) state. Partitions #3 and #4 are not touched. |
| `false` | Partition #2 is marked FAILED in the report. Playbook continues to #3 and #4. At the end the playbook exits non-zero so CI sees the failure, but the report shows what worked. |

## Things that go wrong (and what to do)

| Error | Cause | Fix |
|---|---|---|
| `PCHID 0140 NOT FOUND` (in pre-flight) | typo or adapter not present | check HMC GUI → Adapters → Card Location |
| `port_uri index error` | `port: 1` on a single-port card | set `port: 0` |
| First partition succeeds, second fails on SG create | name collision: SG names must be unique per CPC | use distinct `storage.group_name` per partition |
| WWPN list comes back empty | SG attachment slow to commit | playbook re-fetches; if persistent, check FCP fabric reachability |
| Same NIC name across partitions | NIC names are unique PER partition (good) — playbook works | not actually a problem |
| Hit HMC API rate limit | concurrent requests | bump `inter_partition_pause` to 10–15 s |

## Add a new partition later

Just append another entry to `vars/partitions.yml`:

```yaml
  - name: linuxone-app03
    type: linux
    ifl_processors: 4
    initial_memory: 16384
    network_adapters:
      - { name: nic0, pchid: "0140", port: 2, vlan_id: 120, description: "..." }
    storage:
      group_name: "sg-app03"
      connectivity: 2
      boot_volume: { name: "vol-boot-app03", size: 50 }
      data_volumes: [ { name: "vol-data-app03-01", size: 200, description: "data" } ]
```

Re-run the same `provision-lpars.yml`. Existing partitions are
**idempotent** — the `zhmc_partition` / `zhmc_nic` / `zhmc_storage_group`
modules with `state: present` see they already exist and report `ok` or
update only what changed. Only the new partition gets actually
provisioned.

## DPM-only

This playbook only works on a CPC running in **Dynamic Partition
Manager (DPM)** mode. The pre-flight check fails fast on classic-mode
CPCs. To check your CPC mode without running the playbook:

```bash
ansible localhost -m ibm.ibm_zhmc.zhmc_cpc \
    -a "hmc_host=hmc.fmr.com hmc_auth='{userid:user,password:pwd}' \
        name=Z16-A01 state=facts" | grep dpm-enabled
```
