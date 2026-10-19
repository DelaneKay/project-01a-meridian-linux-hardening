# Hardening verification evidence

This file summarizes sanitized outputs from successful lab runs and read-only checks. No passwords, private keys, or customer records are included. Customer and report content was synthetic.

## Lab versions

- Target: Ubuntu Server 26.04.1 LTS
- Ansible control node: Ansible Core 2.20.1, Python 3.14.4
- Target Python interpreter discovered by Ansible: `/usr/bin/python3.14`
- Assessment tool: Lynis 3.1.7
- Topology: Windows workstation → WSL SSH client → `mds-admin-01` Ansible control node → isolated VirtualBox network → `mds-ubuntu-server-01` target

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

The mock worker generated a clearly labelled synthetic report, remained active, and passed ordinary-admin read and write denial checks. The idempotent rerun reported `ok=9 changed=0 failed=0 unreachable=0`. See [the saved terminal output](mock-worker-idempotent-run.txt) and [screenshot](mock-worker-idempotent-run.png).

## Evidence limits

The file modes and effective service properties above were checked directly on the target after remediation. The `data-files.yml` task summary is preserved from the successful Ansible run shared during the lab. This evidence demonstrates the tested controls in this isolated VM; it is not a claim that SSH, firewall, auditd, or every Lynis suggestion was remediated.
