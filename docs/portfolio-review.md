# Portfolio review

Reviewed on 2026-10-19 against the repository contents and lab evidence.

## Assessment

This is a solid, appropriately scoped portfolio case study for a junior DevSecOps project. It shows investigation of an existing Linux workload, Ansible remediation, systemd hardening, access checks and a repeat run. The environment was your own Windows laptop running WSL and two Ubuntu VMs locally in VirtualBox: one admin/control VM and one target. That is a valid way to build and demonstrate this project; cloud hosting and clean-VM reproduction are not requirements for the story you tell here.

The work is a fictional lab simulation, not a production deployment or a real compromise. The README should describe it that way.

## Review completed

- Read the README, all three playbooks, inventory example, walkthrough, findings and evidence records.
- Confirmed both linked screenshots are present in the repository tree.
- Checked relative Markdown links in the reviewed files; no missing local targets were found.
- Looked for common private-key and token patterns in current text files; none were found. This was not a full history scan or dedicated secrets scan.
- Checked the repository metadata: it is currently private.
- Did not rerun the Ansible playbooks or perform a new lab assessment. Successful outputs already captured in the repository are the execution evidence.

## Changes made

- Added a .gitignore for live inventory, credential and local runtime files.
- Clarified parent-directory conditions when describing file permissions.
- Described the limits of the access tests and mock worker accurately.
- Updated the main project page to say the lab ran locally on your laptop with VirtualBox and that WSL was the SSH client.

## Optional improvements

These are ideas for a later iteration, not missing requirements for this portfolio project:

| Possible improvement | What it would add |
| --- | --- |
| Check report freshness during every run | Show that a new report and log entry were written during that run, rather than only confirming that a report exists and the service is active. The initial run screenshot already shows the worker producing synthetic output. |
| Test an allowed and denied write inside the systemd sandbox | Add direct evidence that the service can write to its required paths and cannot write elsewhere under ProtectSystem=strict. Existing evidence shows effective systemd settings and successful mock writes. |
| Add a synthetic starting-state setup | Let a reader prepare the fictional workload quickly. This is optional because the project investigates an already provisioned server, as described in its scenario. |
| Add CI syntax/lint checks | Automate checks when the playbooks change. Useful for a growing repository, but not needed to make this small lab portfolio credible. |

The repository does not need a fresh cloud host, auditd, SSH/firewall remediation, or a higher Lynis score to support its current Project 1a claims. Those can be separate projects if they match your learning goals.

## Sharing

The repository is private now. Recruiters will need access unless you choose to make it public. No visibility change was made. Decide on a license only if you want to grant others permission to reuse the work.
