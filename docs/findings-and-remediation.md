# Findings and remediation record

**Environment:** Fictional Meridian Digital Services reporting lab. The server and records are synthetic.  
**Assessment type:** Reactive Linux hardening of an already provisioned service.  
**Status:** Three application and service findings were addressed in the lab. A Lynis sudoers-directory warning was reviewed and left unchanged pending comparison with the system's package and local policy.

## Finding 1 — Any local account could modify the customer file

**Evidence before remediation:** `/srv/mds/records/customers.csv` was root-owned with mode `0666` (`-rw-rw-rw-`). Those permission bits allow any local account to write the existing file, including changing or emptying its contents.

**Risk:** An ordinary local account could tamper with synthetic customer records. This is an integrity risk and could undermine reports that depend on the file.

**Control applied:** Ansible set the file owner and group to `mds-reports` and its mode to `0600`. The dedicated reporting service account retains read/write access; ordinary `mdsadmin` access is denied. A system administrator can still override access through authorized root/sudo administration.

**Verification:** The `harden.yml` access checks passed for service-account read/write and ordinary-admin read/write denial. A later live metadata check confirmed `-rw------- 600 mds-reports:mds-reports`.

## Finding 2 — The reporting service had a broad runtime identity and weak process restrictions

**Evidence before remediation:** The original unit ran as `nobody:nogroup` and had `UMask=0000`, `NoNewPrivileges=no`, `PrivateTmp=no`, `ProtectSystem=no`, and `ProtectHome=no`.

**Risk:** The generic identity did not express the worker's specific access needs. The permissive umask could create overly accessible files, and the unit lacked systemd restrictions that limit privilege gain and filesystem writes.

**Control applied:** Ansible created the non-login `mds-reports` system account and configured the worker to run as that user and group. The unit now uses `NoNewPrivileges=true`, `UMask=0027`, `PrivateTmp=true`, `ProtectSystem=strict`, `ProtectHome=true`, and explicit `ReadWritePaths` for records, logs, and archives.

**Verification:** A live read-only check confirmed the service was active, ran as `mds-reports:mds-reports`, and had the expected effective systemd properties and three writable paths.

## Finding 3 — Log and archive data had broader access than the service required

**Evidence before remediation:** The log directory was root-owned mode `0755`; `application.log` was root-owned mode `0666`; the archive directory was root-owned mode `0777`; and `export.csv` was root-owned mode `0644`.

**Risk:** Other local accounts could read customer exports and, for the world-writable log and archive paths, could change or remove data.

**Control applied:** `data-files.yml` set `/srv/mds/logs` and `/srv/mds/archive` to `mds-reports:mds-reports` mode `0700`; it set `application.log` and `export.csv` to that owner and mode `0600`. A separate systemd drop-in allowed writes to the two required paths under `ProtectSystem=strict`.

**Verification:** The successful playbook run reported all four paths changed to the intended owner and modes. A second run reported `changed=0`. Live metadata checks later confirmed the log and archive directories are mode `0700` and owned by `mds-reports`.

## Lynis observation reviewed but not changed

Lynis 3.1.7 reported `/etc/sudoers.d` as root-owned with mode `0755` and labelled its permissions “possibly unsafe.” The directory's mode grants write access only to its owner (root); group and other write bits are absent. The report did not by itself show that an ordinary account could edit sudo configuration. This project recorded the warning for policy review and did not change the directory, because sudoers permissions need to be checked against the distribution's expected configuration before remediation.

## Scope notes

SSH and firewall hardening, auditd installation, and production application security were outside this lab's completed scope. Lynis helped identify areas for investigation; the project uses direct configuration and access checks as evidence rather than an aggregate score.
