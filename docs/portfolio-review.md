# Portfolio review and remaining work

Reviewed on 2026-10-19 against the repository files and recorded lab evidence.

## Assessment

Suitable as a focused, guided Linux-hardening case study. It demonstrates investigation, least-privilege decisions, Ansible remediation, systemd configuration and repeat-run verification. It is not a complete server-security baseline or a turnkey lab provisioner.

## What was reviewed

- All three published playbooks and the example inventory.
- README, walkthrough, findings record and summarized terminal evidence.
- Repository file tree: both referenced PNG evidence files are present.
- Relative Markdown file links: no missing targets were found in the reviewed files.
- Common private-key/token patterns in current text: no matches found. This was not a dedicated secrets scan and did not inspect all Git history.
- Repository visibility: private at review time.

This review did not execute the playbooks, repeat the lab on clean VMs, or run Ansible lint/syntax checks. Earlier successful lab runs remain the runtime evidence.

## Corrections made during review

- Added a .gitignore for live inventory, credentials and local runtime files. Ignore rules do not remove already tracked files or replace secrets scanning.
- Qualified file-permission claims with parent-directory access requirements.
- Distinguished file-content write restrictions from directory-based deletion/replacement.
- Identified the saved text as an abridged terminal record.
- Made the limits of report verification and service-namespace testing explicit.

## Remaining work, by practical value

| Priority | Item | Why it matters / completion evidence |
| --- | --- | --- |
| Required for public sharing | Decide when to make the repository public or grant a reviewer access. | A private repository cannot be viewed by recruiters using an ordinary link. Visibility was not changed by this review. |
| Strongest next technical improvement | Check that the mock report timestamp or mtime advances and a new log entry appears. | Existing report presence plus active service state can pass without proving current writes. Preserve output of the stronger check. |
| Security evidence gap | Inspect parent directories and test deletion/replacement protection with a disposable fixture. | File mode 0600 alone does not prevent deletion through a writable parent. Do not use the customer file for a destructive test. |
| Security evidence gap | Test an allowed and a forbidden write from inside the actual service sandbox. | Account-only runuser tests do not exercise the systemd mount namespace. |
| Reproducibility improvement | Provide a clearly labelled synthetic starting fixture and validate on clean VMs. | Current playbooks require an existing unit and files. A fresh checkout alone cannot reconstruct the original lab. |
| Automation improvement | Add CI syntax/lint checks, then show a passing run. | Helps catch YAML and automation regressions; no CI success is currently claimed. |

A licensing choice can be made before inviting reuse. No license was added automatically because it determines permissions granted to other users.

## Interview discussion points

- Why was a dedicated non-login identity better than sharing nobody?
- Why does NoNewPrivileges not stop writes to an already world-writable file?
- How do ReadWritePaths and Unix permissions interact?
- Why can an expected access denial appear as FAILED in an ad-hoc command but pass a playbook test?
- What does changed=0 demonstrate, and what does it not demonstrate?
- Which claims require new evidence before being made?

The strongest presentation is precise: explain the tested controls, the remaining risks, and the next improvement. Additional tooling is valuable when it closes a specific gap; it is not required merely to make the repository look larger.
