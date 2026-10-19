# Replicate the Meridian Linux hardening exercise

Follow the stages in order. Each block names the machine where it runs; the explanation beneath it covers every command. Related commands are grouped to reduce repetition. Check the result before moving on.

This is a streamlined replay, not a verbatim shell history. Added replication instructions are identified below; original screenshots remain evidence of our original runs.

## 1. Required starting point

~~~text
Windows → WSL SSH client → mds-admin-01 → mds-ubuntu-server-01
                          Ansible         reporting workload
~~~

Two Ubuntu server VMs separate management from the workload. WSL is the workstation entry point. The admin VM connects to the target over the isolated VirtualBox lab network, reflecting remote server administration.

| Machine | Lab address | Prerequisites |
| --- | --- | --- |
| WSL | Workstation | Git, tar, SSH/SCP and an existing key for the admin VM |
| Admin VM | 192.168.56.101 from WSL | Git, Ansible, SSH access and a key for the target |
| Target VM | 192.168.77.10 from admin | systemd, Python 3, sudo and the existing reporting workload |

The SSH account is **mdsadmin**, with sudo access on the target. Our lab used Ansible Core 2.20.1 and Lynis 3.1.7.

**Replication boundary:** the playbooks remediate an existing server. They do not create VMs, networking, SSH keys, Ansible installation or the original insecure workload. The target must already contain:

- The systemd unit **mds-reports.service**.
- Directories **/srv/mds/records**, **/srv/mds/logs** and **/srv/mds/archive**.
- Synthetic files **customers.csv**, **application.log** and **export.csv** in those respective directories.
- Parent-directory traversal permissions that allow the reporting identity to reach its files.

The hidden provisioning script is not included. Fresh VMs need an equivalent synthetic workload prepared first; this repository does not yet reproduce that initial provisioning.

All three playbooks assert the target IP **192.168.77.10**. If yours differs, update both the inventory and the assertions in every playbook. The account names and paths also describe this specific lab.

## 2. Obtain and transfer Lynis

**In WSL:**

~~~sh
command -v git tar ssh scp
git clone --depth 1 --branch 3.1.7 https://github.com/CISOfy/lynis.git ~/lynis-3.1.7
tar -czf ~/lynis-3.1.7.tar.gz -C ~ lynis-3.1.7
sha256sum ~/lynis-3.1.7.tar.gz
scp -i ~/.ssh/mds_admin_ed25519 ~/lynis-3.1.7.tar.gz mdsadmin@192.168.56.101:~/
ssh -i ~/.ssh/mds_admin_ed25519 mdsadmin@192.168.56.101
~~~

**Why:** command -v locates the required tools. Git fetches release 3.1.7 with limited history. Tar creates one compressed archive; -C makes its paths relative to your home directory. SHA-256 provides a fingerprint for transfer checks. SCP copies it using the specified key; SSH opens the admin session. Substitute your existing key filenames if different.

The exact original packaging command was not preserved; this tar command is a replication equivalent. **Compare your own archive's hash across machines.** A fresh archive may differ from ours because of timestamps or Git metadata. Our original hash was:

~~~text
c0808d6b9e9a784ba912a32b2b43052d57ce0a2a2741735194969a3021ab739e
~~~

**On the admin VM:**

~~~sh
sha256sum ~/lynis-3.1.7.tar.gz
scp -i ~/.ssh/mds_target_ed25519 ~/lynis-3.1.7.tar.gz mdsadmin@192.168.77.10:~/
ssh -i ~/.ssh/mds_target_ed25519 mdsadmin@192.168.77.10
~~~

**Why:** check the admin copy against the WSL fingerprint, transfer it using the target key held on the admin VM, then open the target session.

**On the target:**

~~~sh
sha256sum ~/lynis-3.1.7.tar.gz
~~~

**Checkpoint:** all three hashes for your archive must match before extraction. Matching hashes detect transfer differences; they do not independently authenticate the upstream source.

## 3. Install Lynis and preserve a baseline

**On the target.** This installation block is a reconstruction for a new lab; the original extraction/copy commands were not captured. If **/opt/lynis-3.1.7** already exists from this exercise, inspect it and skip extraction and ownership changes.

~~~sh
tar -tzf ~/lynis-3.1.7.tar.gz
sudo tar --no-same-owner -xzf ~/lynis-3.1.7.tar.gz -C /opt
sudo chown -R root:root /opt/lynis-3.1.7
sudo chmod -R go-w /opt/lynis-3.1.7
ls -ld /opt /opt/lynis-3.1.7 /opt/lynis-3.1.7/include/functions
cd /opt/lynis-3.1.7
./lynis show version
sudo ./lynis audit system --quick
~~~

**Why:** first list the archive and confirm its top-level directory is **lynis-3.1.7/**. The next command extracts it into /opt without retaining archived ownership. Chown explicitly sets root ownership; chmod removes group/other write permission while preserving executable bits. Ls checks metadata. Cd ensures this Lynis release finds its support files; the version command should print 3.1.7. The privileged audit inspects system configuration, with --quick avoiding normal pauses.

After the scan, save its baseline before another audit overwrites the active logs:

~~~sh
sudo install -d -m 0700 /root/mds-audit-baseline
sudo cp --update=none /var/log/lynis.log /var/log/lynis-report.dat /root/mds-audit-baseline/
sudo sha256sum /var/log/lynis.log /root/mds-audit-baseline/lynis.log /var/log/lynis-report.dat /root/mds-audit-baseline/lynis-report.dat
~~~

**Why:** install creates a root-only evidence directory. Copy preserves existing baseline files. SHA-256 verifies each original/saved pair on the first capture. If a baseline already exists, different hashes may mean a newer scan is being compared with older evidence. We originally used cp -n; the spelling above follows the warning printed by that system.

Inspect the workload too:

~~~sh
sudo systemctl cat mds-reports.service
sudo stat -c '%A %a %U:%G %n' /srv/mds/records/customers.csv /srv/mds/logs /srv/mds/logs/application.log /srv/mds/archive /srv/mds/archive/export.csv
~~~

**Why:** systemctl shows the unit and drop-ins. Stat shows symbolic/numeric permissions, ownership and filenames together. This consolidated block replaces several historical checks. Record your own starting state: the scanner does not establish the application's access requirements.

Our starting state included customer records and an application log at **0666**, an archive directory at **0777**, and a service using **nobody:nogroup** with permissive systemd settings. See [findings and remediation](findings-and-remediation.md).

## 4. Prepare the repository and inventory

Return from the target SSH session to the **admin VM**:

~~~sh
exit
~~~

**Why:** configuration automation runs on the control node. Confirm the prompt now shows **mds-admin-01**.

For a new checkout, run on the admin VM:

~~~sh
git clone https://github.com/DelaneKay/project-01a-meridian-linux-hardening.git
cd project-01a-meridian-linux-hardening/ansible
cp inventory/hosts.ini.example inventory/hosts.ini
nano inventory/hosts.ini
~~~

**Why:** clone obtains the playbooks; cd enters their directory; cp makes a working inventory; nano lets you replace **REPLACE_WITH_YOUR_TARGET_KEY** with the target-key filename on the admin VM. Confirm the SSH user and address too. Save with Ctrl+O, Enter; exit with Ctrl+X. Repository access is needed while this repository is private. Never insert private-key contents or passwords into the inventory.

These are new replication setup instructions. Our original files already lived under **~/mds-devsecops/ansible**. Use that directory instead if continuing the original lab; do not overwrite an existing inventory.

Check the controller, connection, privilege escalation and YAML together:

~~~sh
ansible --version
ansible -i inventory/hosts.ini mds-ubuntu-server-01 -m ansible.builtin.ping
ansible -i inventory/hosts.ini mds-ubuntu-server-01 -b --ask-become-pass -m ansible.builtin.command -a "id"
ansible-playbook -i inventory/hosts.ini harden.yml data-files.yml mock-worker.yml --syntax-check
~~~

**Why:** the first command identifies the installed version. Ansible ping tests SSH and remote Python, not ICMP. The privileged id command should show root, proving sudo works. The last command parses all three playbooks before mutation. These connectivity checks are added replication prerequisites, not claimed original terminal captures.

| Option | Purpose |
| --- | --- |
| -i | Select the inventory file. |
| -m | Select an Ansible module for an ad-hoc task. |
| -a | Supply that module's arguments. |
| -b | Become the privileged user, root by default. |
| --ask-become-pass | Prompt for the target sudo password. |
| --syntax-check | Parse the playbooks without applying them; this does not prove runtime success. |

**Checkpoint:** ping returns pong, sudo succeeds, and syntax checks pass.

## 5. Apply all three playbooks in order

**On the admin VM, from the same Ansible directory:**

~~~sh
ansible-playbook -i inventory/hosts.ini harden.yml data-files.yml mock-worker.yml --ask-become-pass
~~~

**Why:** Ansible runs the named playbooks in order with one sudo-password prompt. This groups the separate commands used during development; the YAML files remain separate for review and reuse. Investigate failed/unreachable results before continuing.

| Playbook | What it achieves and why |
| --- | --- |
| [harden.yml](../ansible/harden.yml) | Creates the non-login mds-reports identity, applies systemd limits, gives it ownership of customers.csv at 0600, and tests service-account access and ordinary-admin denial. |
| [data-files.yml](../ansible/data-files.yml) | Restricts log/archive directories to 0700 and their existing files to 0600, owned by mds-reports, and permits these service write paths. |
| [mock-worker.yml](../ansible/mock-worker.yml) | Replaces the sleep infinity placeholder with a Python worker that updates a synthetic report and log every 60 seconds, then checks output, service state and ordinary-admin denial. |

The file module's **state: file** does not create missing data files. That is why the workload prerequisites matter.

| Systemd setting | Purpose and limit |
| --- | --- |
| User/Group=mds-reports | Run with a dedicated identity whose file access can be scoped. |
| NoNewPrivileges=true | Prevent privilege gains through execution, such as setuid programs. It does not remove existing file-write permission. |
| UMask=0027 | Remove group write and all other-user bits from newly created files/directories. It does not change existing modes or guarantee 0600 by itself. |
| PrivateTmp=true | Give the service private /tmp and /var/tmp views. |
| ProtectSystem=strict | Make most filesystem paths read-only in the service's mount namespace. |
| ProtectHome=true | Hide home directories from the service. |
| ReadWritePaths | Allow selected paths within that read-only view; ownership and modes still govern access. |

The two writable-path drop-ins accumulate records, logs and archives because neither resets the list. A handler reloads systemd and restarts the service after configuration changes. The mock-worker playbook flushes its handler before checking output.

## 6. Verify the result and repeat for idempotence

**On the admin VM:**

~~~sh
ansible -i inventory/hosts.ini mds-ubuntu-server-01 -m ansible.builtin.command -a "systemctl show mds-reports.service -p ActiveState -p User -p Group -p UMask -p NoNewPrivileges -p PrivateTmp -p ProtectSystem -p ProtectHome -p ReadWritePaths"
ansible -i inventory/hosts.ini mds-ubuntu-server-01 -b --ask-become-pass -m ansible.builtin.command -a "stat -c '%A %a %U:%G %n' /srv/mds/records/customers.csv /srv/mds/logs /srv/mds/logs/application.log /srv/mds/archive /srv/mds/archive/export.csv /srv/mds/archive/mock-report.txt"
ansible-playbook -i inventory/hosts.ini harden.yml data-files.yml mock-worker.yml --ask-become-pass
~~~

**Why:** systemctl reads effective settings after combining all drop-ins. Privileged stat can inspect files inside the private directories. The repeat playbook run checks convergence: when the desired configuration already exists, Ansible should report **changed=0** and avoid configuration-triggered restarts.

Expected results:

- Active service, user/group mds-reports, UMask=0027, protections enabled and records/logs/archive in ReadWritePaths.
- Customer, log, export and mock-report files at 0600; log/archive directories at 0700; ownership mds-reports.
- Automated checks allow reporting-account access and deny ordinary mdsadmin access to customer records and the mock report.
- A labelled synthetic report, no failed/unreachable tasks, and no configuration changes on repetition.

Ad-hoc command tasks can display CHANGED for read-only queries because the module cannot infer their intent. The playbook verification tasks explicitly use **changed_when: false**.

Our separately executed mock-worker runs reported **ok=10 changed=3** initially and **ok=9 changed=0** later. Combined-run task counts will differ; validate outcomes rather than matching those counts.

**What this proves:** the mock writes to the archive and log inside the hardened service. Customer-file runuser tests check account permissions outside that service namespace; the mock does not process customer records. Mode 0600 alone also does not prevent deletion by someone with write/search permission on the parent directory.

## 7. Preserve evidence and describe the scope

Keep the first successful mock-worker output and the later unchanged run. Our evidence is:

- [First-run screenshot](../evidence/mock-worker-run.png): deployment, report generation and denial checks.
- [Repeat-run screenshot](../evidence/mock-worker-idempotent-run.png): successful verification with changed=0.
- [Verification record](../evidence/hardening-verification.md) and [findings record](findings-and-remediation.md): supporting state and decisions.

You can repeat the target-side Lynis audit from stage 3 after preserving the baseline. Compare relevant findings, rather than treating its broad score as the only success measure. Do not claim every finding was remediated.

This is reactive service/data hardening in a fictional lab. SSH/firewall hardening, auditd and a production reporting application were not completed here. A separate cloud project is planned for proactive hardening.

## Optional troubleshooting from the original exercise

| Symptom | Explanation and next action |
| --- | --- |
| dpkg-query -W lynis finds no package | It checks package-manager installation; our manually cloned tree was not a registered package. |
| Missing include directory or ./lynis not found | Change into /opt/lynis-3.1.7 before running it. The ./ prefix refers to the current directory. |
| Lynis flags user-owned files | We cancelled with Ctrl+C and used a root-owned installation. Stage 3 provides replay preparation. |
| Missing sudo password or runuser rejects non-root execution | For privileged ad-hoc tasks, supply both -b and --ask-become-pass. The playbooks already set become: true. |
| Python interpreter discovery warning | It reports remote Python selection. It did not cause our successful runs to fail; investigate if that environment changes. |
| Rerun only service settings | Add --tags service_runtime to a harden.yml invocation. Use the full initial run so account/data prerequisites are not skipped. |

**Individual permission test, on the admin VM:**

~~~sh
ansible -i inventory/hosts.ini mds-ubuntu-server-01 -b --ask-become-pass -m ansible.builtin.command -a "runuser -u mds-reports -- test -w /srv/mds/records/customers.csv"
~~~

**Why:** runuser changes identity, then test -w checks write access without modifying contents. Success is silent with code 0. Substituting mdsadmin should return 1 for denied access; an ad-hoc task calls that failed. The playbook negative checks explicitly require code 1. A privileged administrator still has sudo/root override.

**Process identity, on the target:**

~~~sh
systemctl show mds-reports.service -p MainPID -p User -p Group
~~~

**Why:** this retrieves the current PID and configured identity. Substitute that PID into **ps -p PID -o pid,euser,egroup,args** to see the process's effective user, group and command. Our original PID 42826 was temporary and must not be reused blindly.

**Lynis file-ownership investigation, on the target:**

~~~sh
sudo find /opt/lynis-3.1.7 \( ! -user root -o ! -group root -o -perm /022 \) -print
ls -l /opt/lynis-3.1.7/db/languages/{en-GB,nl-BE,de-AT,br,nl-NL,en-US}
~~~

**Why:** find locates unexpected ownership or group/other write bits. Our results included language symlinks; ls showed their targets. Linux symlink modes commonly display 777 and do not control access to their targets. Add **-type f** after the search path if you specifically want regular files.

**Sudoers-directory investigation, on the target:**

~~~sh
ls -ld /etc/sudoers.d
sudo grep -n -A 12 -B 4 'possibly unsafe permissions' /var/log/lynis.log
~~~

**Why:** ls shows ownership/mode; grep finds warnings with surrounding log lines. This grouped diagnostic reconstructs how to inspect the excerpts discussed in our lab. Our root-owned directory was 0755; the inspected Lynis test expected no other-user permission bits. We documented the warning without changing it. A scanner warning warrants investigation; it does not itself prove that ordinary users can modify sudo policy.
