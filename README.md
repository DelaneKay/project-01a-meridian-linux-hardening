# Project 1a / 11 — The Meridian reporting server

**An existing server. An undisclosed fault. A controlled investigation.**

Meridian Digital Services is fictional. This project follows my investigation of an existing Ubuntu reporting server in a two-VM VirtualBox lab. I looked for unsafe access, assessed the risk, automated focused fixes, and checked that the reporting service still worked. All customer records and reports were synthetic.

> **Portfolio claim:** I assessed an existing Linux service with native tools and Lynis, used Ansible to restrict its identity and data access, and verified both the allowed and denied paths with a synthetic reporting worker.

## The three-part lab

```text
My Windows laptop
├── Windows PowerShell / OpenSSH ── SCP induction ZIP ──┐
├── WSL / OpenSSH ──────────────── Lynis transfer ──────┤
└── VirtualBox                                        ↓
    ├── mds-admin-01: Ansible control node
    │   ├── Adapter 2 host-only: 192.168.56.101
    │   └── Adapter 3 Internal Network mds-sec-lab (archive expects .77.20; guest IP unverified)
    └── mds-ubuntu-server-01: target
        └── Adapter 2 Internal Network mds-sec-lab (inventory host: 192.168.77.10)
```

I ran the lab locally on my Windows laptop. VirtualBox hosted the two Ubuntu Server VMs: `mds-admin-01` and the target. WSL ran on the same laptop; I used it for SSH and to transfer Lynis. It was not a third VM. I used Windows PowerShell's OpenSSH `scp` command to copy the induction ZIP from Downloads to the admin VM. Ansible and the inventory lived on the admin VM, while the reporting service and synthetic data lived on the target.

The written adapter record documents the settings, but does not establish guest routing or complete egress isolation; all addresses are lab-only.

## The scenario and the role of AI

The scenario pointed to an AI-generated package called `mds-induction.zip` as the source of the target's unsafe setup. I had not examined it yet, so I did not know what it had changed. Finding out became the focus of my investigation. This was a controlled lab with synthetic data, not a real breach.

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
- [VirtualBox network settings](evidence/virtualbox-network.md): written summary of the adapter topology and its evidence limits.
- The published playbooks were copied from `mds-admin-01` as separate YAML files after their SHA-256 checksums were matched. The earlier preflight-only draft remains outside this repository.

## Step-by-step walkthrough

The [project journey](docs/step-by-step-lab-walkthrough.md) is a first-person account of how I set up the local VirtualBox lab, transferred the induction ZIP, investigated the existing workload with Lynis and Linux tools, applied Ansible changes, and checked the results. It explains the security principle behind each decision, how it addressed the Meridian scenario, and what the evidence supports. It also maps the work to ISC2 CC, Google Cybersecurity Certificate, and AWS Security Specialty learning topics, while making clear this was a local Linux lab rather than an AWS deployment or proof of certification.

## Repository map

```text
README.md                         This case study, architecture and run instructions
.gitignore                        Excludes live inventory and local credential files
docs/findings-and-remediation.md  Evidence-to-control record
docs/step-by-step-lab-walkthrough.md First-person investigation, decisions, commands and verification
evidence/                         Verification output, mock-worker screenshots and network notes
lab-assets/mds-induction.zip       AI-assisted induction archive copied to the admin VM
lab-assets/mds-induction.sha256   SHA-256 checksum for the unchanged archive
ansible/                          Final playbooks copied from the admin server
  harden.yml                      Reporting identity, systemd and customer file
  data-files.yml                  Log and archive permissions and writable paths
  mock-worker.yml                 Synthetic workload and runtime checks
  inventory/hosts.ini.example     Safe inventory example without credentials
```

The inventory example uses placeholders; it contains no private keys, passwords, or live host details. I did not include real customer data or credentials in the repository. I copied the three playbooks from `mds-admin-01` and checked their SHA-256 hashes against the source files. The unchanged `mds-induction.zip` archive is also included, with its checksum. Its deployment workflow is designed to create an insecure lab setup, so it belongs only in this controlled exercise and should never be run against a real server.


## Run the playbooks

The remediation playbooks expect the reporting service and synthetic data to already exist on the target. The repository includes the induction ZIP I copied to the admin VM, but does not include VirtualBox setup or SSH configuration. The [project journey](docs/step-by-step-lab-walkthrough.md#2-i-established-ssh-access-and-transferred-the-induction-package) explains what I did and what I could verify. I ran the playbooks from the `ansible` directory on the admin VM and kept the live inventory and private key out of Git.

Run the playbooks in this order:

```sh
# Run from this repository's ansible directory.
ansible-playbook -i inventory/hosts.ini harden.yml data-files.yml mock-worker.yml --syntax-check
ansible-playbook -i inventory/hosts.ini harden.yml data-files.yml mock-worker.yml --ask-become-pass
```

The first playbook creates the restricted service identity, applies systemd limits, protects the customer record file, and checks who can access it. The second permits the service's log and archive paths and locks down their ownership and modes. The third installs the synthetic reporting worker, switches the systemd unit to it, and checks the generated report, service state, and ordinary-admin access. `--ask-become-pass` makes Ansible prompt for the sudo password instead of storing it in the command or inventory.

## Scope and limits

This is **Project 1a: reactive Linux hardening**. Lynis supported the baseline investigation, but its aggregate score is not used as the measure of success; it covers many checks outside this exercise. The direct permission, systemd, service and idempotence evidence supports the narrower claims above. SSH provided the management connection, but sshd and firewall hardening, auditd and a production reporting application were **not** completed in this project.

The synthetic worker demonstrates archive and log writes in this lab. Customer-file checks use the account outside the service's filesystem namespace; the worker does not process customer data. The current mock verification checks report existence and service activity, not report freshness on every run. Parent-directory deletion permissions and out-of-scope write denial inside the service have not been demonstrated by the published tests.

The repository contains the investigation notes, Ansible changes, service and access checks, and mock-worker run evidence from this project.
