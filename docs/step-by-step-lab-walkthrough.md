# My Meridian Linux hardening investigation

This document records my Project 1a journey: how I investigated an existing reporting server, what I changed with Ansible, and how I checked that the reporting workload still worked. It is a retrospective account, not a set of instructions for a reader to follow. Command blocks show commands from the work where they were captured; reconstructed examples are identified as such.

## The situation at Meridian Digital Services

Meridian Digital Services is a fictional company. Its reporting worker reads customer records and creates reports, logs, and exports. The Ubuntu server hosting that service was already running when my investigation began; I was not building a clean server from scratch.

The scenario presented the existing target as compromised through an AI-generated package called `mds-induction.zip`. I had not examined the package yet, so I did not know what changes it had made. Discovering those changes became the investigation. This was a controlled VirtualBox lab with synthetic data, not a real breach.

The security principles that guided my investigation were network segmentation, integrity, least privilege, confidentiality, and availability. Segmentation limits which systems can directly reach one another and can reduce the blast radius of a mistake or compromise. Integrity checks help reveal whether transferred tools or evidence changed unexpectedly. Least privilege and confidentiality guided the service identity and file permissions, while availability shaped my checks that the reporting service still ran after each change.

My toolkit was VirtualBox for the two-server lab; Windows PowerShell, WSL, OpenSSH, and SCP for management and file transfer; Lynis and native Linux commands for investigation; and Ansible with systemd for remediation and verification. I used SHA-256 checksums to compare transferred files. The steps below explain the principle behind each security decision, how it applied to Meridian, and what my evidence supports.

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

**Principle — network segmentation and defense in depth.** Keeping the control node and target on a VirtualBox Internal Network limits direct lab traffic to VMs attached to that named network. The admin VM also had a host-only interface so I could connect from the Windows host. This arrangement helped separate host-to-admin management from admin-to-target lab traffic and limited accidental exposure through a bridged adapter. It reduces reachability and potential lateral movement; it does not by itself prove complete isolation, because guest routes, forwarding, and other VirtualBox sharing features also matter.

The [VirtualBox adapter notes](../evidence/virtualbox-network.md) record the settings behind this layout. They show the admin VM using Adapter 2 as a host-only adapter and Adapter 3 on mds-sec-lab; Adapter 1 (NAT) is disabled. The host-only address used to reach the admin was 192.168.56.101. The target settings record Adapter 1 (NAT) disabled and Adapter 2 on the same internal network; Adapters 3 or 4 were not documented, so I cannot claim those were disabled. The target address 192.168.77.10 is in the Ansible inventory. The value 192.168.77.20 appears as the control-node address expected by the induction archive, but the adapter record does not verify that guest address.



This follows the ISC2 CC ideas of network security architecture and segmentation, and the Google Cybersecurity Certificate's network-security and Linux topics. The AWS Security Specialty also covers infrastructure security and network protections; here I applied the underlying security reasoning to VirtualBox and Ubuntu, not to AWS services.

The package's single-adapter assumption conflicts with the recorded settings
The adapter record covers only the settings captured; it does not establish every adapter or VirtualBox setting, or prove guest-level routing and forwarding state.

## 2. I established SSH access and transferred the induction package

**Principle — secure administration and authentication.** SSH encrypts the management session and file transfer, protecting credentials and data in transit. SSH host keys let a client check that it is connecting to the expected server; client keys identify the administrator. Separate keys for the two hops also limit which connection each credential can authorize. The key names visible in later transfers were `mds_admin_ed25519` for WSL-to-admin transfers and `mds_target_ed25519` for admin-to-target transfers. The key-creation commands were not captured, so I am not presenting reconstructed `ssh-keygen` commands as if I ran them. The saved evidence does not show an independent fingerprint check at the first host-key prompt, so I cannot claim that verification step was documented.

The exact induction-package transfer shown in the screenshot was run in **Windows PowerShell**:

~~~powershell
scp "$env:USERPROFILE\Downloads\mds-induction.zip" mdsadmin@192.168.56.101:~/
~~~

PowerShell expanded `$env:USERPROFILE` to my Windows user folder. SCP read the ZIP from Downloads and copied it into the admin account's home directory over SSH. On the first connection, I had to address the host-key prompt; the admin account password was used when requested. This specific transfer did not use WSL's `/mnt/c` path. The encrypted SCP channel protected the archive in transit, while the checksum comparison provided a separate check that the received bytes matched the recorded copy.

The file's SHA-256 fingerprint was:

~~~text
f23cb7326c3fb42868286a72705d7f25cef80cb896d8ff1739193d94f0e18c07
~~~

**Principle — integrity.** Integrity means protecting information and software from unauthorized or accidental alteration. A SHA-256 digest gives me a compact fingerprint to compare copies: matching hashes across transfer points support the conclusion that the bytes were unchanged between those checks. A hash by itself does not prove who created the archive or that its contents are safe; that would require a trusted source, signature, or independently trusted expected hash.

I used the hash to identify the exact archive and check that the copy had not changed. The same archive is preserved in this repository at [lab-assets/mds-induction.zip](../lab-assets/mds-induction.zip), with its fingerprint in [lab-assets/mds-induction.sha256](../lab-assets/mds-induction.sha256). Its `START-HERE.md` says the package had not been deployed. I preserve the archive unchanged as the file transferred to the admin VM, not as proof that it provisioned the workload observed on the target.

The package contains `lab.py`, `session.json`, and `assets/session.py`. Its deployment asset is designed to seed fictional records, logs, archives, and a deliberately permissive reporting service for the investigation scenario. The package documented these stages on the admin server:

~~~sh
python3 lab.py preflight --ask-become-pass
python3 lab.py deploy --ask-become-pass
python3 lab.py verify --ask-become-pass
~~~

The package documentation describes preflight checks for control-node identity, Ansible and inventory details, network state, and target SSH/sudo readiness. Its deploy mode is intended to seed and verify the synthetic starting workload through Ansible; verify checks the expected initial state. These are documented package commands, not confirmed deployment output from my target. The complete output for a successful seed stage was not retained.

The package also describes a pre-deployment VirtualBox snapshot named `01-Pre-Vulnerable-Lab` and a rollback procedure. These remain package instructions, not evidence that deployment occurred. The adapter mismatch and deployment uncertainty are recorded in the preceding section.

## 3. I moved Lynis into the investigation

I used Lynis 3.1.7 to help identify areas to inspect. In WSL, I cloned the tagged source, created an archive, recorded its hash, and transferred it to the admin server. I then transferred it from the admin server to the target. The captured commands included:

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

**Principle — integrity of privileged tools.** A privileged audit tool must come from files that ordinary users cannot alter. Otherwise, root could execute tampered code. Lynis stopped because the source tree was owned by my ordinary account. I cancelled that run with Ctrl+C, inspected `/opt` and the installation, and made the installed tree root-owned and not writable by group or other users before continuing. The exact extraction and ownership-changing commands were not preserved; the final listings showed the Lynis files under `/opt/lynis-3.1.7` owned by `root:root`. That mattered because running a privileged audit over user-controlled program files would be unsafe.

**Principle — preserve evidence integrity and access control.** Baseline output can be overwritten by later scans, and audit logs may contain sensitive system details. I saved the Lynis log and report in a root-only directory before later runs could replace them:

~~~sh
sudo install -d -m 0700 /root/mds-audit-baseline
sudo cp -n /var/log/lynis.log /var/log/lynis-report.dat /root/mds-audit-baseline/
sudo sha256sum /var/log/lynis.log /root/mds-audit-baseline/lynis.log /var/log/lynis-report.dat /root/mds-audit-baseline/lynis-report.dat
~~~

The directory was root-only, limiting access to the baseline evidence. Matching source and saved hashes confirmed that the saved copies matched at capture time; as with the archive hashes, this check proves equality of the compared bytes, not their original source or authenticity. The `cp -n` command printed a portability warning, but completed; its suggested spelling was `--update=none`.

**Principle — risk-based assessment.** A scanner is a source of leads, not a replacement for checking the system state and judging which risks matter to the service. Lynis gave me a list of suggestions, but its score did not move much after the focused fixes. That did not mean the work had no value: I had addressed specific, evidenced application-service risks, while many general host recommendations were outside this project. I used Lynis as an investigative aid and relied on the actual permission, service, and Ansible checks to demonstrate the selected improvements.

## 4. I traced the findings to actual permissions and service settings

I examined the service definition and the files it could access. The evidence showed:

- `/srv/mds/records/customers.csv` was owned by root with mode `0666`. Those mode bits allowed local users who could traverse the parent directories to alter its contents.
- `/srv/mds/logs` was `0755`, and `application.log` was `0666`.
- `/srv/mds/archive` was `0777`, and `export.csv` was `0644`.
- The original service ran as `nobody:nogroup` with `UMask=0000`, `NoNewPrivileges=no`, `PrivateTmp=no`, `ProtectSystem=no`, and `ProtectHome=no`.

**Principle — evidence-based risk assessment.** I needed to understand the effective permissions and service configuration before selecting controls; a finding matters when its reachable attack path and impact are clear. These checks explained why the problem mattered. A writable customer file could be tampered with, and broad log/archive access could expose or alter reporting artifacts. The generic service identity also made it harder to give the worker only the permissions it required.

**Principle — protect privileged configuration while validating findings.** Sudo policy controls administrative access, so its files must be writable only by trusted administrators. Lynis raised a separate warning about `/etc/sudoers.d` being root-owned with mode `0755`. I inspected the directory and the related Lynis log excerpt. The mode allowed root to write; it did not show that an ordinary account could change sudo policy. I documented the warning and left it unchanged pending comparison with the system's expected package and local policy rather than applying an unverified permission change.

## 5. I organized the Ansible remediation as separate playbooks

**Principle — controlled, repeatable change.** Security configuration should be reviewable and consistently applied, so I used Ansible to declare the intended state and kept distinct control areas in separate YAML files. This supports repeatability and helps reduce manual configuration drift. I kept the work in separate YAML files so each control area remained reviewable and could be documented separately:

- [harden.yml](../ansible/harden.yml) created the non-login `mds-reports` system account, configured the systemd identity and restrictions, changed ownership/mode on `customers.csv`, and checked allowed and denied access.
- [data-files.yml](../ansible/data-files.yml) restricted the log and archive directories and files, and allowed the reporting service to write to those paths under `ProtectSystem=strict`.
- [mock-worker.yml](../ansible/mock-worker.yml) replaced the placeholder `sleep infinity` process with a synthetic worker and checked its generated report, service state, and ordinary-admin denial.

Before applying changes, I checked that Ansible could parse the YAML and reach the target. The syntax check did not change the target; the ping tested SSH and remote Python, and the privileged identity check verified sudo:

~~~sh
ansible-playbook -i inventory/hosts.ini harden.yml --syntax-check
ansible -i inventory/hosts.ini mds-ubuntu-server-01 -m ansible.builtin.ping
ansible -i inventory/hosts.ini mds-ubuntu-server-01 -b --ask-become-pass -m ansible.builtin.command -a "id"
~~~

**Principle — scope control and change safety.** A correct hardening change applied to the wrong server can still cause an incident. The playbooks asserted that the target was `192.168.77.10` before making changes. That check reduced the risk of applying the remediation to the wrong host. Each playbook used privilege escalation because it had to manage a system account, systemd drop-ins, or protected files.

**Principle — least privilege.** A service should run with only the identity and access it needs, rather than sharing a generic or administrator identity. Ansible created `mds-reports` as a system account with no home directory and a non-login shell. A systemd drop-in then set `User=mds-reports` and `Group=mds-reports`. This gave the workload a dedicated identity whose file permissions could be scoped to its purpose.

**Principle — defense in depth.** File permissions limit what the account can access, while systemd restrictions add a separate boundary around what the process can do. These controls reduce the impact of a compromised worker or coding error; they do not replace correct ownership, application security, or monitoring. I next limited the service's runtime authority. The drop-in set:

| Setting | Why I used it |
| --- | --- |
| `NoNewPrivileges=true` | Prevents the service from gaining privileges through execution, such as through setuid programs. |
| `UMask=0027` | Removes group-write and other-user permissions from newly created files/directories by default. It does not change existing permissions. |
| `PrivateTmp=true` | Gives the service a private view of `/tmp` and `/var/tmp`. |
| `ProtectSystem=strict` | Makes most filesystem paths read-only in the service's mount namespace. |
| `ProtectHome=true` | Hides home directories from the service. |
| `ReadWritePaths` | Allows the service to write only to its required records, logs, and archive paths within the protected filesystem view. |

These restrictions complemented Unix ownership and modes; they did not replace them. In particular, `NoNewPrivileges` does not prevent writing a file when its permissions already allow the service to write it.

**Principles — confidentiality, integrity, and least privilege.** Customer data should be readable only by an authorized identity, and unauthorized writers should not be able to alter its contents. For customer records, Ansible changed `customers.csv` from root-owned `0666` to `mds-reports:mds-reports` with mode `0600`. The reporting account retained read/write access; ordinary `mdsadmin` could not read or write it. I made this choice because the worker needed routine access, while authorized administrators still needed a sudo/root override.

**Principles — confidentiality, integrity, and least privilege.** Logs and exports can reveal sensitive information and may be used to investigate activity, so unnecessary access or modification weakens both privacy and trust in the records. The data-files playbook similarly set the log and archive directories to `0700`, and their existing log/export files to `0600`, owned by `mds-reports`. The systemd writable-path settings allowed the service to use those directories while `ProtectSystem=strict` made the rest of the filesystem read-only to it.

## 6. I verified the worker and access controls

**Principle — verify both security and availability.** A hardening change is useful only if it enforces the intended access policy and the legitimate workload remains usable. I checked the allowed path, denied path, and service state instead of treating a successful playbook exit as sufficient proof.

I did not treat a successful Ansible run alone as proof. The service remained active and systemd reported `User=mds-reports` and `Group=mds-reports`. I also checked the running process with `ps`; before the mock worker replaced it, the process was `/usr/bin/sleep infinity` running under the dedicated account.

The access checks tested both sides of the policy. The reporting account could read and write the customer file. Ordinary `mdsadmin` could not read or write it. The checks were run through `runuser` with Ansible privilege escalation, because a non-root account cannot use `runuser` to switch to another user. In the playbooks, an expected permission-denied return code counted as a successful negative test.

The systemd property checks later showed `UMask=0027`, `NoNewPrivileges=yes`, `PrivateTmp=yes`, `ProtectSystem=strict`, `ProtectHome=yes`, and writable paths for records, logs, and archives. The directories were owned by `mds-reports` at `0700`; the customer, log, export, and mock report files were at `0600`.

To prove the service still performed useful work, I replaced the placeholder with the mock worker. It wrote a clearly labelled synthetic report and log entry. The Ansible output showed the service active, the synthetic report timestamp, and successful checks that ordinary `mdsadmin` could not read or write the mock report. The mock worker exercised the archive and log paths; it did not process customer records.

I also reran the playbooks. The final mock-worker verification run reported `changed=0`, showing Ansible found the configuration already in the desired state. Earlier runs showed changes while I was still adding controls; those were expected as the playbooks evolved.

The verification has limits. A `0600` file prevents ordinary users from opening it directly, but deletion or replacement also depends on permissions on its parent directory. The published checks confirmed read/write permissions on the file, not deletion behavior. The mock worker did not test forbidden writes from inside every systemd isolation boundary, and report-existence checks alone do not prove a fresh write when an older report is already present.

## 7. I preserved the evidence and recorded scope

**Principle — accountability and evidence preservation.** Keeping the output of checks makes the work reviewable and supports later comparison. I saved two terminal screenshots: the first successful mock-worker run and the later idempotent run. The repository also contains a verification record and a findings/remediation record:

- [Initial mock-worker run](../evidence/mock-worker-run.png)
- [Idempotent mock-worker rerun](../evidence/mock-worker-idempotent-run.png)
- [Hardening verification record](../evidence/hardening-verification.md)
- [Findings and remediation record](findings-and-remediation.md)

The work completed here was reactive Linux service and data hardening on a fictional workload in a local VirtualBox lab. Lynis helped identify areas to investigate, but I used direct evidence to select and verify specific controls. I did not harden sshd settings or firewall rules, install auditd, or build a production reporting application. SSH provided administration and file transfer; SSH and firewall hardening were outside this project's scope.

## Learning principles and course references

I mapped the controls in this project to concepts covered by these learning resources. This is a principle crosswalk for a local Linux lab; it does not mean I used AWS services or that this project proves a certification.

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
| Lynis flagged user-owned files | I cancelled the privileged scan, checked the installation ownership, and placed the tool under root ownership in `/opt`. |
| An ad-hoc Ansible task reported “Missing sudo password” | It needed both privilege escalation (`-b`) and a become-password prompt (`--ask-become-pass`). |
| `runuser` refused to switch users | The Ansible command was not privileged. The working test used `-b --ask-become-pass`. |
| The Python interpreter discovery warning appeared | Ansible had selected `/usr/bin/python3.14`; the warning did not prevent the successful runs. |
| A rerun still showed changes | I had just added or changed directives. After the configuration settled, the next run reported `changed=0`. |

The Lynis sudoers-directory warning was another useful investigation lesson. The scanner reported `/etc/sudoers.d` as possibly unsafe because it was mode `0755`. Checking ownership and actual write bits showed that only root could write to it. I recorded the warning and did not alter the system based solely on the scanner message.
