# Project 1a / 11 — The Meridian reporting server

**An existing server. An undisclosed fault. A controlled investigation.**

Meridian Digital Services is fictional. This project documents a reactive DevSecOps exercise on an isolated Ubuntu reporting server that was already running when the investigation began. The goal was to discover unsafe access, explain the risk, automate proportionate fixes, and prove the reporting workload still ran. All customer records and report output in the lab were synthetic.

> **Portfolio claim:** I assessed an existing Linux service with native tools and Lynis, used Ansible to restrict its identity and data access, and verified both the allowed and denied paths with a synthetic reporting worker.

## The three-part lab

```text
My Windows laptop
├── Windows PowerShell / OpenSSH ── SCP induction ZIP ──┐
├── WSL / OpenSSH ──────────────── Lynis transfer ──────┤
└── VirtualBox                                        ↓
    ├── mds-admin-01: Ansible control node
    │   ├── Host-only adapter: 192.168.56.101
    │   └── Internal Network mds-sec-lab (archive expects .77.20; guest IP unverified)
    └── mds-ubuntu-server-01: target
        └── Internal Network "mds-sec-lab": 192.168.77.10
```

I built and ran this entire lab locally on my laptop using VirtualBox for the **two Ubuntu server VMs**: the admin server and the target. WSL ran on the same Windows laptop and was used for the Lynis transfer and SSH work; it was not a third VM. The induction ZIP was copied from the Windows Downloads folder to the admin VM using Windows PowerShell's OpenSSH `scp` command. The admin VM held the Ansible inventory and ran the playbooks against the target VM. The target hosted the reporting service and synthetic data.

This topology placed target-side traffic on a named VM-to-VM segment, while the admin VM also had a host-only management adapter. The screenshots document adapter attachments but do not establish guest routing or complete egress isolation; all addresses are lab-only.

## The scenario and the role of AI

The target was **not a fresh server awaiting a standard build**. The exercise centered on a deliberately insecure reporting workload whose planted findings were withheld during the blind assessment. I copied an AI-assisted induction package from Windows to the admin server. Its own `START-HERE.md` says it had not yet been deployed, and its preflight requires exactly one active adapter on the admin VM. The VirtualBox screenshots in this repository show two enabled admin adapters, so the transferred ZIP cannot be treated as verified proof of how the observed vulnerable state was created. I investigated the state that was present on the target; the exact provisioning command or package version that placed it there is not established by the saved evidence. This was a controlled simulation, not evidence of a real-world intrusion or malicious AI attack.

AI also helped draft and explain parts of the remediation. I checked the commands, permissions, service state, and Ansible results in the lab. The evidence below distinguishes what was observed from what is merely planned.

## Investigation → control → proof

| Stage | What I did | Why it mattered |
| --- | --- | --- |
| 1. Establish a baseline | Inspected the service, process identity, file modes, ownership and Lynis findings. | A scanner points to areas to investigate; the operating system's actual state determines the risk. |
| 2. Trace access | Checked who could read or write the customer file, logs and exported report. | World-writable or world-readable paths could expose or change synthetic customer information. |
| 3. Create a service identity | Used Ansible to create `mds-reports` as a non-login system account and run the worker under it. | The workload no longer depended on a generic identity; its permissions could be scoped to its job. |
| 4. Limit the service | Applied `NoNewPrivileges`, `UMask=0027`, `PrivateTmp`, `ProtectHome` and `ProtectSystem=strict`, with explicit writable paths. | Systemd restrictions reduced the service's ability to gain privileges or write outside its required directories. |
| 5. Restrict data | Set `customers.csv` and the report/log files to owner-only `0600`; set the log and archive directories to `0700`, owned by `mds-reports`. | The worker retained required access while an ordinary account could not read or alter those files. |
| 6. Exercise the controls | Replaced the placeholder `sleep infinity` process with a synthetic worker that produced a mock report and log entry. | A running service alone would not show that the hardened worker could still perform its intended write operations. |
| 7. Re-run and verify | Checked service activity, expected access denials, generated output and a second Ansible run with `changed=0`. | The controls were testable and the automation converged without repeatedly changing the target. |

### A concrete before/after

The lab began with `/srv/mds/records/customers.csv` owned by root and mode `0666`: its mode permitted writing by any local user able to traverse the parent directories. Ansible changed it to `mds-reports:mds-reports` and `0600`. Verification showed the reporting identity could read and write, while ordinary `mdsadmin` access checks were denied. Administrative override remained possible through controlled sudo/root access.

The original reporting unit also used permissive settings such as `UMask=0000`, `NoNewPrivileges=no`, and no systemd filesystem protections. The Ansible drop-in changed the effective service settings while keeping it active. `ReadWritePaths` allowed the worker's records, logs and archive locations under `ProtectSystem=strict`.

## Evidence

### Initial run

The first mock-worker playbook run installed and started the synthetic worker, produced its report, and passed both ordinary-admin denial checks. It completed with `ok=10 changed=3 failed=0`.

![First successful mock-worker run](evidence/mock-worker-run.png)

### Idempotent rerun

The second run checked the report and access controls again with `changed=0`, showing Ansible found the target already in the requested state. An abridged terminal record is available [here](evidence/mock-worker-idempotent-run.txt).

![Idempotent mock-worker rerun](evidence/mock-worker-idempotent-run.png)

- [Hardening verification record](evidence/hardening-verification.md): captured lab versions, current service properties, file permissions, and successful playbook summaries.
- [Findings and remediation record](docs/findings-and-remediation.md): evidence, risk, control, verification, and scope decisions for each finding.
- [VirtualBox network settings](evidence/virtualbox-network.md): screenshots and explanation of the host-only and internal-network layout.
- The published playbooks were copied from `mds-admin-01` as separate YAML files after their SHA-256 checksums were matched. The earlier preflight-only draft remains outside this repository.

## Step-by-step walkthrough

The [project journey](docs/step-by-step-lab-walkthrough.md) is a first-person account of how I set up the local VirtualBox lab, transferred the induction ZIP, investigated the existing workload with Lynis and Linux tools, applied Ansible changes, and checked the results. It explains why I took each step, distinguishes captured commands from details that were reconstructed or not preserved, and notes that the transferred ZIP is not verified as the package that seeded the target.

## Repository map

```text
README.md                         This case study, architecture and run instructions
.gitignore                        Excludes live inventory and local credential files
docs/findings-and-remediation.md  Evidence-to-control record
docs/step-by-step-lab-walkthrough.md First-person investigation, decisions, commands and verification
evidence/                         Verification output, mock-worker screenshots and VirtualBox settings
lab-assets/mds-induction.zip       AI-assisted induction archive copied to the admin VM
lab-assets/mds-induction.sha256   SHA-256 checksum for the unchanged archive
ansible/                          Final playbooks copied from the admin server
  harden.yml                      Reporting identity, systemd and customer file
  data-files.yml                  Log and archive permissions and writable paths
  mock-worker.yml                 Synthetic workload and runtime checks
  inventory/hosts.ini.example     Safe inventory example without credentials
```

The inventory example uses placeholders rather than keys, passwords or the live inventory file. No real customer data, private keys or credentials belong in this portfolio repository. The three playbooks were copied from `mds-admin-01`; SHA-256 checksums matched the source files. `lab-assets/mds-induction.zip` is the unchanged induction archive copied from Windows to the admin VM over SSH/SCP. Its SHA-256 is recorded beside it. The archive says it had not been deployed, and its one-adapter preflight conflicts with the saved admin settings, so this repository does not claim that it seeded the target. Its deploy mode is designed to create insecure lab state; treat it as a lab-only artifact and do not run it against a real server.


## Run the playbooks

The remediation playbooks expect the reporting service and synthetic data to exist on the target. The repository preserves the induction ZIP that I copied to the admin VM, but the archive says it was not deployed, and its preflight assumptions conflict with the saved VirtualBox adapter settings. The [project journey](docs/step-by-step-lab-walkthrough.md#2-i-established-ssh-access-and-transferred-the-induction-package) records that uncertainty. The repository does not create VirtualBox VMs or configure their adapters or SSH access. To review or rerun the remediation, I used the `ansible` directory on the admin/control node and kept the live inventory and private key out of Git.

Run the playbooks in this order:

```sh
# Run from this repository's ansible directory.
ansible-playbook -i inventory/hosts.ini harden.yml data-files.yml mock-worker.yml --syntax-check
ansible-playbook -i inventory/hosts.ini harden.yml data-files.yml mock-worker.yml --ask-become-pass
```

The first playbook creates the restricted service identity, applies systemd limits, protects the customer record file, and checks who can access it. The second permits the service's log and archive paths and locks down their ownership and modes. The third installs the synthetic reporting worker, switches the systemd unit to it, and checks the generated report, service state, and ordinary-admin access. `--ask-become-pass` makes Ansible prompt for the sudo password instead of storing it in the command or inventory.

These playbooks change the lab target. Read each task before running it, confirm the inventory points at the intended isolated VM, and keep access to the VM console available for recovery.

## Scope and limits

This is **Project 1a: reactive Linux hardening**. Lynis supported the baseline investigation, but its aggregate score is not used as the measure of success; it covers many checks outside this exercise. The direct permission, systemd, service and idempotence evidence supports the narrower claims above. SSH provided the management connection, but sshd and firewall hardening, auditd and a production reporting application were **not** completed in this project. Project 1b is planned as a separate, proactive cloud-server hardening exercise.

The synthetic worker demonstrates archive and log writes in this lab. Customer-file checks use the account outside the service's filesystem namespace; the worker does not process customer data. The current mock verification checks report existence and service activity, not report freshness on every run. Parent-directory deletion permissions and out-of-scope write denial inside the service have not been demonstrated by the published tests.

The evidence in this repository records the investigation, Ansible changes, service checks, access checks and mock-worker runs completed in this project.
