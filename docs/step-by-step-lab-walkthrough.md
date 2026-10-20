# My Meridian Linux hardening investigation

- **Core technologies:** Linux, Lynis, Ansible, and systemd.
- **Problem addressed:** I investigated an existing reporting server presented as compromised and found unsafe service and data access settings to correct.
- **Outcome:** I used Ansible to apply repeatable, idempotent hardening. The reporting account kept the access it needed, ordinary `mdsadmin` was denied, and a later playbook run reported `changed=0`.

This is my account of Project 1a: investigating an existing reporting server, making changes with Ansible, and checking that the service still worked. Commands are shown as I used them when they were captured; anything reconstructed is labelled.

## The situation at Meridian Digital Services

Meridian Digital Services is a fictional company. Its reporting worker reads customer records and creates reports, logs, and exports. The Ubuntu server hosting that service was already running when my investigation began; I was not building a clean server from scratch.

The scenario presented the existing target as compromised through an AI-generated package called `mds-induction.zip`. I had not examined the package yet, so I did not know what changes it had made. Discovering those changes became the investigation. This was a controlled VirtualBox lab with synthetic data, not a real breach.

I kept a few security fundamentals in mind as I worked. Network segmentation limits which systems can reach one another and can contain the impact of a mistake or compromise. Integrity checks help show whether transferred files changed. Least privilege and confidentiality guided the service account and file permissions, and availability mattered when I checked that the service continued to run.

I used VirtualBox for the lab, PowerShell and WSL with OpenSSH/SCP for administration and file transfers, Lynis and Linux commands to investigate, and Ansible with systemd to make and verify changes. I also compared SHA-256 checksums to check file integrity. The walkthrough explains how each tool and principle fit into the investigation.

## 1. I built the lab on my Windows laptop

I ran two Ubuntu Server virtual machines locally in VirtualBox. One was the admin/control server, where I kept Ansible and ran playbooks. The other was the target reporting server. WSL was also available on the same laptop and I used it for SSH and the Lynis transfer; it was not a third VM. The induction ZIP itself was copied from Windows PowerShell, as shown by the command captured in the project conversation.

~~~text
Windows laptop
├── Windows PowerShell / OpenSSH ── SCP induction ZIP ──┐
├── WSL / OpenSSH ──────────────── Lynis transfer ──────┤
└── VirtualBox                                        ↓
    ├── mds-admin-01: Ansible control server
    │   ├── Adapter 1: NAT disabled
    │   ├── Adapter 2: Host-only, 192.168.56.101
    │   └── Adapter 3: Internal Network mds-sec-lab (archive expects .77.20; guest IP unverified)
    └── mds-ubuntu-server-01: target
        ├── Adapter 1: NAT disabled
        └── Adapter 2: Internal Network mds-sec-lab (target inventory address: .77.10)
~~~

I connected the admin VM and target through VirtualBox's Internal Network, `mds-sec-lab`, and used a host-only adapter to reach the admin VM from Windows. This is network segmentation: keeping the lab's VM-to-VM traffic on a separate network reduces direct exposure and limits how far a problem could spread. It reflects the defense-in-depth idea that one boundary should support, rather than replace, the others.

The [VirtualBox adapter notes](../evidence/virtualbox-network.md) record the settings behind this layout. They show the admin VM using Adapter 2 as a host-only adapter and Adapter 3 on mds-sec-lab; Adapter 1 (NAT) is disabled. The host-only address used to reach the admin was 192.168.56.101. The target settings record Adapter 1 (NAT) disabled and Adapter 2 on the same internal network; Adapters 3 or 4 were not documented, so I cannot claim those were disabled. The target address 192.168.77.10 is in the Ansible inventory. The value 192.168.77.20 appears as the control-node address expected by the induction archive, but the adapter record does not verify that guest address.


These choices connect to network security and segmentation topics in the ISC2 CC and Google Cybersecurity Certificate. The AWS Security Specialty covers infrastructure security as well. I applied those general principles to a VirtualBox lab; I did not use AWS services here.

## 2. I established SSH access and transferred the induction package

SSH gave me an encrypted way to administer the VMs and transfer files. I used separate keys for the two connections: `mds_admin_ed25519` from WSL to the admin VM and `mds_target_ed25519` from the admin VM to the target. On the first connection I handled the host-key prompt, then authenticated with the admin account. I did not preserve the key-generation commands, so I have left them out rather than reconstructing them.

The exact induction-package transfer shown in the screenshot was run in **Windows PowerShell**:

~~~powershell
scp "$env:USERPROFILE\Downloads\mds-induction.zip" mdsadmin@192.168.56.101:~/
~~~

PowerShell expanded `$env:USERPROFILE` to my Windows user folder, and SCP copied the ZIP from Downloads to the admin VM over SSH. When prompted, I entered the admin account password. I ran this transfer in PowerShell, so I did not use WSL's `/mnt/c` path. SSH encrypted the transfer; the checksum comparison gave me a separate way to check that the file arrived unchanged.

The file's SHA-256 fingerprint was:

~~~text
f23cb7326c3fb42868286a72705d7f25cef80cb896d8ff1739193d94f0e18c07
~~~

I checked the archive's integrity by comparing its SHA-256 fingerprint at the transfer points. A matching hash is evidence that the copies contained the same bytes at the time I checked them; it does not prove who created the archive or whether it was safe. I recorded the fingerprint in [lab-assets/mds-induction.sha256](../lab-assets/mds-induction.sha256), and the archive is preserved at [lab-assets/mds-induction.zip](../lab-assets/mds-induction.zip).

The package contains `lab.py`, `session.json`, and `assets/session.py`. Its deployment asset is designed to seed fictional records, logs, archives, and a deliberately permissive reporting service for the investigation scenario. The package documented these stages on the admin server:

~~~sh
python3 lab.py preflight --ask-become-pass
python3 lab.py deploy --ask-become-pass
python3 lab.py verify --ask-become-pass
~~~

The package documentation says that `preflight` checks the admin VM, Ansible, inventory, network, and target SSH/sudo access. Its `deploy` stage is intended to create the synthetic starting workload, while `verify` checks the expected setup. The package also recommends taking a VirtualBox snapshot named `01-Pre-Vulnerable-Lab` before deployment. I did not keep a successful deployment transcript, so these commands and steps describe what the package was designed to do, rather than a complete record of output from my target.

## 3. I moved Lynis into the investigation

Lynis 3.1.7 was installed and run on the isolated target VM, `mds-ubuntu-server-01`. I used WSL to clone the tagged source, create an archive, record its hash, and send it to `mds-admin-01`; from there, I transferred it to the target. The admin VM was the Ansible control server, not the machine being audited. The captured transfer commands included:

~~~sh
git clone --depth 1 --branch 3.1.7 https://github.com/CISOfy/lynis.git ~/lynis-3.1.7
sha256sum ~/lynis-3.1.7.tar.gz
scp -i ~/.ssh/mds_admin_ed25519 ~/lynis-3.1.7.tar.gz mdsadmin@192.168.56.101:~/
~~~

On the admin server, I checked the hash and used the target-specific key for the second hop:

~~~sh
sha256sum ~/lynis-3.1.7.tar.gz
scp -i ~/.ssh/mds_target_ed25519 ~/lynis-3.1.7.tar.gz mdsadmin@192.168.77.10:~/
~~~

The hash recorded for that archive was `c0808d6b9e9a784ba912a32b2b43052d57ce0a2a2741735194969a3021ab739e`. Comparing hashes across the hops helped catch transfer errors and gave me evidence of file integrity at those points. The original archive creation command was not captured, so the repository walkthrough describes a representative packaging command separately rather than claiming it as an exact historical command.

My first attempt to run Lynis from the home-directory copy failed because it could not find its include directory. I moved the installation under `/opt/lynis-3.1.7`, checked that its support files were present, and confirmed the version:

~~~sh
cd /opt/lynis-3.1.7
./lynis show version
~~~

The command printed `3.1.7`. The initial audit command was:

~~~sh
sudo ./lynis audit system --quick
~~~

On the isolated target VM `mds-ubuntu-server-01`, the first quick audit stopped because `/home/mdsadmin/lynis-3.1.7/include/functions` was owned by `mdsadmin`. Lynis required that file to be owned by root before running with elevated privileges. I cancelled the scan, then used a root-owned Lynis installation under `/opt/lynis-3.1.7`. The later directory listing showed the installation and its key files owned by `root:root`. I did not preserve the exact commands used to prepare that installation.

I saved the Lynis log and report in a root-only directory before running another scan. This protected potentially sensitive audit output from ordinary users and preserved the baseline for later comparison:

~~~sh
sudo install -d -m 0700 /root/mds-audit-baseline
sudo cp -n /var/log/lynis.log /var/log/lynis-report.dat /root/mds-audit-baseline/
sudo sha256sum /var/log/lynis.log /root/mds-audit-baseline/lynis.log /var/log/lynis-report.dat /root/mds-audit-baseline/lynis-report.dat
~~~

The directory was accessible only to root. Matching hashes showed that the saved files were identical to the originals at the time of the copy. The copy command printed a portability warning, but it completed successfully.

I treated Lynis as a source of leads, then checked the server itself before deciding what to fix. Its score changed little after my focused work because it includes many host-level checks outside this project. That did not erase the improvements I had made to the reporting service and its data; I verified those directly through file permissions, service settings, and Ansible results.

### I checked active services and listening ports

Before inspecting the data files, I checked which services were running and which TCP or UDP ports were listening. This gave me a view of the server's active attack surface and helped me decide what to investigate further. I did not preserve the exact port listing, so I do not name specific ports as findings.

I used these checks to see what was active and listening:

~~~sh
systemctl list-units --type=service --state=running
sudo ss -tulpn
~~~

The first command lists active systemd services. In the second, `ss` displays socket information: `-t` and `-u` include TCP and UDP, `-l` limits the list to listening sockets, `-p` shows the owning process when available, and `-n` keeps addresses and port numbers numeric. The exact command history and port output were not preserved, so these are reconstructed examples and I do not claim specific ports from them.

One result stood out: `mds-reports.service` was active, but its `ExecStart` was `/usr/bin/sleep infinity`, a placeholder that did no reporting work. The base unit also declared `nobody:nogroup`, `UMask=0000`, and no systemd restrictions such as `NoNewPrivileges`, `PrivateTmp`, or filesystem protections. That combination seemed odd for a service described as a reporting worker, so I inspected the unit and the files it was meant to use. Those checks led to the findings in the next section.

## 4. I traced the findings to actual permissions and service settings

I examined the service definition and the files it could access. The evidence showed:

- `/srv/mds/records/customers.csv` was owned by root with mode `0666`. Those mode bits allowed local users who could traverse the parent directories to alter its contents.
- `/srv/mds/logs` was `0755`, and `application.log` was `0666`.
- `/srv/mds/archive` was `0777`, and `export.csv` was `0644`.
- The base service unit declared `User=nobody`, `Group=nogroup`, `UMask=0000`, and disabled systemd restrictions. The later drop-in set the effective service identity to `mds-reports`.

I checked the effective permissions and service configuration before choosing controls. The world-writable customer file could be changed by local users who could reach it, broad log and archive permissions exposed or put reporting files at risk, and the generic service identity made it harder to grant the worker only the access it needed.

Sudo policy controls who can administer the server, so I checked Lynis's warning about `/etc/sudoers.d` carefully. The directory was root-owned with mode `0755`. I inspected the directory and the related Lynis log excerpt. The mode allowed root to write; it did not show that an ordinary account could change sudo policy. I documented the warning and left it unchanged pending comparison with the system's expected package and local policy rather than applying an unverified permission change.

## 5. I automated the remediation with Ansible

By this point, I knew what needed to change: the reporting worker needed a dedicated service identity, customer and report files needed tighter access, and the service needed limits on the filesystems and privileges it could use. I also needed to check that these changes did not stop the worker from running.

I chose Ansible because it was already available on my admin/control VM and could manage the isolated target over SSH. Its playbooks let me describe the desired configuration in YAML, apply it remotely, and check the result without repeating each change by hand.

Automation is a core DevSecOps practice because it turns security requirements into repeatable, reviewable changes. That reduces manual mistakes and configuration drift, and makes it easier to apply the same controls consistently. Automation does not decide whether a control is appropriate or prove that it works; I still had to understand the risk, choose the settings, and verify the outcome.

I split the Ansible work across separate YAML files so I could review each area on its own and rerun the changes consistently:

- [harden.yml](../ansible/harden.yml) created the non-login `mds-reports` system account, configured the systemd identity and restrictions, changed ownership/mode on `customers.csv`, and checked allowed and denied access.
- [data-files.yml](../ansible/data-files.yml) restricted the log and archive directories and files, and allowed the reporting service to write to those paths under `ProtectSystem=strict`.
- [mock-worker.yml](../ansible/mock-worker.yml) replaced the placeholder `sleep infinity` process with a synthetic worker and checked its generated report, service state, and ordinary-admin denial.

Before applying changes, I checked that Ansible could parse the YAML and reach the target. The syntax check did not change the target; the ping tested SSH and remote Python, and the privileged identity check verified sudo:

~~~sh
ansible-playbook -i inventory/hosts.ini harden.yml --syntax-check
ansible -i inventory/hosts.ini mds-ubuntu-server-01 -m ansible.builtin.ping
ansible -i inventory/hosts.ini mds-ubuntu-server-01 -b --ask-become-pass -m ansible.builtin.command -a "id"
~~~

Before changing anything, each playbook checked that the target was `192.168.77.10`. This guard helped prevent me from applying the remediation to the wrong VM. The playbooks used privilege escalation because they needed to manage a system account, systemd drop-ins, or protected files.

I started with the principle of least privilege: the reporting worker needed its own limited identity, not a generic account. Ansible created `mds-reports` as a system account with no home directory and a non-login shell. A systemd drop-in then set `User=mds-reports` and `Group=mds-reports`. This gave the workload a dedicated identity whose file permissions could be scoped to its purpose.

I added systemd restrictions as another layer alongside file permissions. They limit what the process can do if the worker is compromised or behaves unexpectedly. The drop-in set:

| Setting | Why I used it |
| --- | --- |
| `NoNewPrivileges=true` | Prevents the service from gaining privileges through execution, such as through setuid programs. |
| `UMask=0027` | Removes group-write and other-user permissions from newly created files/directories by default. It does not change existing permissions. |
| `PrivateTmp=true` | Gives the service a private view of `/tmp` and `/var/tmp`. |
| `ProtectSystem=strict` | Makes most filesystem paths read-only in the service's mount namespace. |
| `ProtectHome=true` | Hides home directories from the service. |
| `ReadWritePaths` | Allows the service to write only to its required records, logs, and archive paths within the protected filesystem view. |

These restrictions complemented Unix ownership and modes; they did not replace them. In particular, `NoNewPrivileges` does not prevent writing a file when its permissions already allow the service to write it.

For the customer file, I applied confidentiality and least-privilege principles: only the reporting identity should have routine access, and other local accounts should not be able to change the records. Ansible changed `customers.csv` from root-owned `0666` to `mds-reports:mds-reports` with mode `0600`. The reporting account retained read/write access; ordinary `mdsadmin` could not read or write it. I made this choice because the worker needed routine access, while authorized administrators still needed a sudo/root override.

I applied the same access principles to logs and exports. They can contain sensitive details and help explain what happened, so unnecessary reading or changes would weaken both confidentiality and trust in the records. The data-files playbook set the log and archive directories to `0700` and their files to `0600`, owned by `mds-reports`. The systemd writable-path settings allowed the service to use those directories while `ProtectSystem=strict` made the rest of the filesystem read-only to it.

## 6. I verified the worker and access controls

I checked that the reporting service retained the access it needed, ordinary users were denied access, and the service remained active. A successful Ansible run alone would not prove those things, so I checked the systemd settings, file permissions, and actual access results.

Systemd reported `User=mds-reports` and `Group=mds-reports`. It also showed `UMask=0027`, `NoNewPrivileges=yes`, `PrivateTmp=yes`, `ProtectSystem=strict`, `ProtectHome=yes`, and the expected writable paths for records, logs, and archives. The directories were owned by `mds-reports` with mode `0700`; the customer, log, export, and mock report files were `0600`. Before I installed the mock worker, `ps` showed the placeholder `/usr/bin/sleep infinity` process running under the dedicated account.

The access checks confirmed that `mds-reports` could read and write the customer file, while ordinary `mdsadmin` could do neither. I ran these checks through `runuser` with Ansible privilege escalation, because an unprivileged account cannot use `runuser` to switch to another user. The playbooks treat the expected permission-denied result as a successful check.

To prove the service still performed useful work, I replaced the placeholder with the mock worker. It wrote a clearly labelled synthetic report and log entry. The Ansible output showed the service active, the synthetic report timestamp, and successful checks that ordinary `mdsadmin` could not read or write the mock report. The mock worker exercised the archive and log paths; it did not process customer records.

I also reran the playbooks. The final mock-worker verification run reported `changed=0`, showing Ansible found the configuration already in the desired state. Earlier runs showed changes while I was still adding controls; those were expected as the playbooks evolved.

The verification has limits. A `0600` file prevents ordinary users from opening it directly, but deletion or replacement also depends on permissions on its parent directory. The published checks confirmed read/write permissions on the file, not deletion behavior. The mock worker did not test forbidden writes from inside every systemd isolation boundary, and report-existence checks alone do not prove a fresh write when an older report is already present.

## 7. I preserved the evidence and recorded scope

I saved two terminal screenshots so the checks could be reviewed later: the first successful mock-worker run and the later rerun, where Ansible reported no further changes. The repository also contains a verification record and a findings/remediation record:

- [Initial mock-worker run](../evidence/mock-worker-run.png)
- [Idempotent mock-worker rerun](../evidence/mock-worker-idempotent-run.png)
- [Hardening verification record](../evidence/hardening-verification.md)
- [Findings and remediation record](findings-and-remediation.md)

The work completed here was reactive Linux service and data hardening on a fictional workload in a local VirtualBox lab. Lynis helped identify areas to investigate, but I used direct evidence to select and verify specific controls. I did not harden sshd settings or firewall rules, install auditd, or build a production reporting application. SSH provided administration and file transfer; SSH and firewall hardening were outside this project's scope.

## Learning principles behind the work

The controls I used connect to topics in these learning resources. I applied the ideas to a local Linux lab; I did not use AWS services in this project, and the work is not a claim of certification.

| Learning resource | Principles I applied in this project |
| --- | --- |
| [ISC2 Certified in Cybersecurity (CC) exam outline](https://www.isc2.org/certifications/cc/cc-certification-exam-outline) | Confidentiality, integrity, availability; least privilege and access control; network security architecture, segmentation, and defense in depth. |
| [Google Cybersecurity Certificate](https://grow.google/certificates/cybersecurity/) | Linux security, network security, security controls, and using evidence to investigate and reduce risk. |
| [AWS Certified Security – Specialty exam guide](https://docs.aws.amazon.com/aws-certification/latest/security-specialty-03/security-specialty-03.html) | Identity and access management, infrastructure security, data protection, logging and monitoring, and balancing controls with workload requirements. |

The specific implementations here were Linux permissions, Ansible, systemd, VirtualBox networking, and local command-line checks. The principles transfer across platforms, while AWS implementations would use AWS services and the AWS shared-responsibility model.

## Troubleshooting I encountered

| What I saw | What it meant in my investigation |
| --- | --- |
| `dpkg-query -W lynis` found no package | Lynis was cloned and installed manually, so it was not registered with the package manager. |
| Lynis could not find its include directory | I was invoking it outside the untarred installation directory. Running it from `/opt/lynis-3.1.7` resolved the path issue. |
| Lynis stopped the quick audit because `include/functions` was owned by `mdsadmin` | I cancelled the scan on the Ubuntu target and used the root-owned installation under `/opt/lynis-3.1.7`. |
| An ad-hoc Ansible task reported “Missing sudo password” | It needed both privilege escalation (`-b`) and a become-password prompt (`--ask-become-pass`). |
| `runuser` refused to switch users | The Ansible command was not privileged. The working test used `-b --ask-become-pass`. |
| The Python interpreter discovery warning appeared | Ansible had selected `/usr/bin/python3.14`; the warning did not prevent the successful runs. |
| A rerun still showed changes | I had just added or changed directives. After the configuration settled, the next run reported `changed=0`. |

The Lynis sudoers-directory warning was another useful investigation lesson. The scanner reported `/etc/sudoers.d` as possibly unsafe because it was mode `0755`. Checking ownership and actual write bits showed that only root could write to it. I recorded the warning and did not alter the system based solely on the scanner message.
