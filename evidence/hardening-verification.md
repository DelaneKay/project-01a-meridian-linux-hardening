# Hardening verification evidence

This file collects sanitized output from successful lab runs and read-only checks. It contains no passwords, private keys, or real customer records; all customer and report content was synthetic.

## Lab versions

- Target: Ubuntu Server 26.04.1 LTS
- Ansible control node: Ansible Core 2.20.1, Python 3.14.4
- Target Python interpreter discovered by Ansible: `/usr/bin/python3.14`
- Assessment tool: Lynis 3.1.7
The recorded settings place admin Adapter 3 and target Adapter 2 on `mds-sec-lab`; Adapter 1 (NAT) is disabled on both VMs. The inventory lists the target as `192.168.77.10`. The induction archive expects the admin VM at `192.168.77.20`, but the saved adapter settings do not confirm that address. Target Adapters 3 and 4 were not documented. More detail is in the [VirtualBox adapter notes](virtualbox-network.md).

## After `harden.yml`

The Ansible playbook's access checks passed:

- `mds-reports` could read `customers.csv`.
- `mds-reports` could write `customers.csv`.
- `mdsadmin` could not read the file.
- `mdsadmin` could not write the file.

The live read-only metadata check returned:

```text
-rw-------:600:mds-reports:mds-reports:/srv/mds/records/customers.csv
```

The live systemd check returned:

```text
ActiveState=active
UMask=0027
User=mds-reports
Group=mds-reports
ReadWritePaths=/srv/mds/logs /srv/mds/archive /srv/mds/records
PrivateTmp=yes
ProtectHome=yes
ProtectSystem=strict
NoNewPrivileges=yes
```

## After `data-files.yml`

The first run changed the intended permissions and ownership:

```text
/srv/mds/logs                 mds-reports:mds-reports 0700
/srv/mds/logs/application.log mds-reports:mds-reports 0600
/srv/mds/archive              mds-reports:mds-reports 0700
/srv/mds/archive/export.csv   mds-reports:mds-reports 0600
```

The playbook recap reported `ok=7 changed=6 failed=0 unreachable=0`. A later run reported `ok=6 changed=0 failed=0 unreachable=0`, confirming Ansible found the target already in the desired state.

## Mock worker

The mock worker generated a clearly labelled synthetic report, remained active, and passed ordinary-admin read and write denial checks. The idempotent rerun reported `ok=9 changed=0 failed=0 unreachable=0`. See [the abridged terminal record](mock-worker-idempotent-run.txt) and [screenshot](mock-worker-idempotent-run.png).

## Evidence limits

The linked text record omits terminal formatting and the interpreter warning; use the screenshots for the captured display. Report existence and service activity are checked, but freshness on each run is not asserted. Customer-file access checks run outside the service's mount namespace, and parent-directory deletion protection is not tested.

This evidence demonstrates the controls I tested in the local VirtualBox lab.
