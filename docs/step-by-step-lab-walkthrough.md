# Step-by-step lab walkthrough

This is a record of the Meridian Linux hardening exercise from lab preparation through evidence collection. Meridian and its customer records are fictional; all report content was synthetic. The two Ubuntu VMs were an admin/control node and a target. WSL was the workstation's SSH client to the admin VM.

## Lab layout

```text
Windows workstation
└── WSL (SSH client)
    └── mds-admin-01 (Ansible control node)
        └── SSH on the isolated VirtualBox lab network
            └── mds-ubuntu-server-01 (reporting service target)
```

Keeping the control node separate from the workload mirrors how an engineer or automation runner manages remote hosts. It means Ansible and its inventory live on a management system, while SSH connects to the target. The isolated VirtualBox network limits exposure while practicing changes that would affect a real service.

## 1. Check the WSL tools and obtain Lynis

From WSL, check that the tools used to obtain the source are available:

```sh
command -v git curl
```

The recorded output resolved both commands under `/usr/bin`. Clone the fixed Lynis release used for the assessment:

```sh
git clone --depth 1 --branch 3.1.7 https://github.com/CISOfy/lynis.git ~/lynis-3.1.7
```

The `--depth 1` option fetches only the selected revision's recent history, and `--branch 3.1.7` selects the version assessed in the lab.

Prepare and checksum the archive before transfer:

```sh
tar -czf ~/lynis-3.1.7.tar.gz -C ~ lynis-3.1.7
sha256sum ~/lynis-3.1.7.tar.gz
```

The clone, checksum and transfers were recorded in the lab. The exact original archive-creation command was not preserved; the `tar` line above is a reproducible equivalent. The recorded SHA-256 was:

```text
c0808d6b9e9a784ba912a32b2b43052d57ce0a2a2741735194969a3021ab739e
```

Transfer the archive to the admin VM using the existing SSH key:

```sh
scp -i ~/.ssh/mds_admin_ed25519 ~/lynis-3.1.7.tar.gz mdsadmin@192.168.56.101:~/
```

On `mds-admin-01`, compare its checksum with the WSL result. Then transfer it to the target with the target's existing key:

```sh
sha256sum ~/lynis-3.1.7.tar.gz
scp -i ~/.ssh/mds_target_ed25519 ~/lynis-3.1.7.tar.gz mdsadmin@192.168.77.10:~/
```

On the target, compare the checksum again before extracting or using the archive. Matching hashes confirm that the same archive arrived at each machine.

## 2. Run Lynis on the existing target

The first version check was attempted from the wrong path and returned “can't find include directory.” Lynis must run from its extracted program directory so it can find its support files. The working directory used for the assessment was:

```sh
cd /opt/lynis-3.1.7
./lynis show version
sudo ./lynis audit system --quick
```

The version command returned `3.1.7`. The first privileged scan from a user-owned copy stopped at Lynis's ownership safety check. We moved to a root-owned installation under `/opt/lynis-3.1.7`, then ran the scan from there. This guard exists to avoid root executing files a less-privileged user could alter.

Lynis helped identify areas to inspect. Its score was not used as the project's success measure: the work focused on direct findings, the resulting configuration, and access tests.

## 3. Preserve the baseline report

The scan created `/var/log/lynis.log` and `/var/log/lynis-report.dat`. The recorded copy command was:

```sh
sudo cp -n /var/log/lynis.log /var/log/lynis-report.dat /root/mds-audit-baseline/
```

GNU `cp` printed a portability warning for `-n`. For a new run, use its suggested spelling:

```sh
sudo cp --update=none /var/log/lynis.log /var/log/lynis-report.dat /root/mds-audit-baseline/
```

Compare source and saved copies:

```sh
sudo sha256sum /var/log/lynis.log /root/mds-audit-baseline/lynis.log /var/log/lynis-report.dat /root/mds-audit-baseline/lynis-report.dat
```

The hash pairs matched in the lab. The baseline was kept separate so later changes would not overwrite the original assessment evidence.

## 4. Check the Ansible playbook before applying it

On `mds-admin-01`, work from the Ansible directory and syntax-check the service hardening playbook:

```sh
cd ~/mds-devsecops/ansible
ansible-playbook -i inventory/hosts.ini harden.yml --syntax-check
```

The syntax check returned `playbook: harden.yml`. It checks that Ansible can parse the YAML; it does not connect to the target or prove that tasks work.

The inventory points to the intended target at `192.168.77.10`. Each playbook also asserts the target address and SSH connection before continuing.

## 5. Create the service identity and restrict customer records

Run the first remediation playbook:

```sh
ansible-playbook -i inventory/hosts.ini harden.yml --ask-become-pass
```

`--ask-become-pass` prompts for the sudo password when Ansible needs administrative rights. The password is entered at the prompt and is not stored in the command or repository.

The playbook creates `mds-reports` as a non-login system account, configures the systemd service to run under it, adds process and filesystem restrictions, and changes `customers.csv` from world-writable `0666` to owner-only `0600`. It then tests that the reporting identity can read and write the file and that ordinary `mdsadmin` access is denied.

The output showed the service identity, override and handler tasks completed successfully. The later access-check run reported each positive and negative check as `ok`.

## 6. Restrict logs and archive files

Run the separate data-files playbook:

```sh
ansible-playbook -i inventory/hosts.ini data-files.yml --ask-become-pass
```

This playbook adds the log and archive paths to the service's allowed writable paths under `ProtectSystem=strict`. It gives the reporting account ownership of both directories with mode `0700` and protects the log and export files with mode `0600`.

The first run changed the requested paths. A later run reported `changed=0`, showing that Ansible found them already in the requested state.

## 7. Install and verify the synthetic reporting worker

Run the mock-worker playbook:

```sh
ansible-playbook -i inventory/hosts.ini mock-worker.yml --ask-become-pass
```

The playbook installs a small Python worker that writes a clearly labelled synthetic report and a log entry, configures systemd to run it, waits for output, checks the service is active, and confirms ordinary `mdsadmin` cannot read or write the report. The original placeholder service command was `/usr/bin/sleep infinity`; the playbook replaces it with this lab worker.

The first run finished with `ok=10 changed=3 failed=0`. The second run finished with `ok=9 changed=0 failed=0`. Both outputs and screenshots are saved under `evidence/`.

## 8. Verify the effective service and file state

These read-only Ansible commands were used from the control node:

```sh
ansible -i inventory/hosts.ini mds-ubuntu-server-01 -m ansible.builtin.command -a "systemctl is-active mds-reports.service"
ansible -i inventory/hosts.ini mds-ubuntu-server-01 -m ansible.builtin.command -a "systemctl show mds-reports.service -p User -p Group -p UMask -p NoNewPrivileges -p PrivateTmp -p ProtectSystem -p ProtectHome -p ReadWritePaths"
ansible -i inventory/hosts.ini mds-ubuntu-server-01 -m ansible.builtin.command -a "stat -c %A:%a:%U:%G:%n /srv/mds/records/customers.csv /srv/mds/logs /srv/mds/archive"
```

The service returned `active`. The effective properties showed `User=mds-reports`, `Group=mds-reports`, `UMask=0027`, `NoNewPrivileges=yes`, `PrivateTmp=yes`, `ProtectSystem=strict`, `ProtectHome=yes`, and writable paths limited to records, logs and archives. The customer file was `600`; log and archive directories were `700`, all owned by `mds-reports`.

To run a `runuser` access check through Ansible, the remote command needs administrative privilege because `runuser` is restricted to root on this target. Include `-b --ask-become-pass`; without it, the command either failed with “may not be used by non-root users” or Ansible asked for a missing become password. A successful `test -r` or `test -w` prints no text; use its return code: `0` means allowed, `1` means the permission check was denied.

## 9. Record findings and publish the portfolio evidence

The findings record explains the risk, control, and verification for the customer file, service runtime, and data paths. It also records Lynis's `/etc/sudoers.d` warning as reviewed but unresolved; the directory was root-owned and not writable by ordinary accounts, so it was not changed without checking the expected system policy.

The repository contains the three separate playbooks, a credential-free inventory example, the findings record, sanitized command output and both mock-worker screenshots. No SSH private keys, passwords, live inventory, customer data, or hidden provisioning script are included. Project 1a covers reactive Linux hardening; SSH and firewall changes, auditd, and the proactive cloud-server work remain outside this project's completed scope.
