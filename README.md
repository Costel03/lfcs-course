# LFCS Preparation — Linux Administration, Intermediate to Advanced

A complete course for the **Linux Foundation Certified System Administrator (LFCS)**
exam, written to also make you a genuinely capable administrator. Every topic has worked
examples, every chapter ends with a quiz and a hands-on lab, and every lab task has a
step that proves your answer — because the exam marks the *state of the machine*, not
what you meant to do.

**Author:** Costel Iacob · **Language:** English · **Level:** intermediate → advanced ·
**Time:** 85–100 hours, most of it in the labs.

---

## The exam

| | |
|---|---|
| Format | Online, proctored, **performance-based** — real tasks in a real terminal |
| Tasks | 17–20 |
| Duration | 2 hours |
| Pass mark | 67% |
| Attempts | 2 included (one free retake) |
| Valid for | 2 years |
| Environment | Linux hosts reached with `ssh <host>`; `sudo -i` available. Has been Ubuntu since the 2023 refresh — **confirm the current version on the Linux Foundation site before booking** |
| Allowed help | `man` pages, `/usr/share/doc`, packages from the distribution. **No web browser.** |

Source: [LFCS official page](https://training.linuxfoundation.org/certification/linux-foundation-certified-sysadmin-lfcs/)
and the candidate instructions, checked 2026-10-03. Objectives are revised from time to
time — compare the domain list below with the official page before you start.

The no-browser rule shapes this whole course: every chapter teaches you to find answers
in `man` and `/usr/share/doc`, because that is all you will have.

---

## Exam domains

| Domain | Weight | Chapters |
|---|---|---|
| Operations Deployment | 25% | 05, 06, 07, 12, 13, 14 |
| Networking | 25% | 10, 11 |
| Storage | 20% | 08, 09 |
| Essential Commands | 20% | 01, 02, 03, 05, 06, 11 |
| Users and Groups | 10% | 04 |

---

## Syllabus

| # | Chapter | Hours | Domain |
|---|---|---|---|
| 01 | Introduction and the shell — [notes](notes/01-introduction.md) · [quiz](quizzes/01-introduction.md) · [lab](labs/01-introduction.md) | 5 | Essential Commands |
| 02 | The file system — [notes](notes/02-filesystem.md) · [quiz](quizzes/02-filesystem.md) · [lab](labs/02-filesystem.md) | 6 | Essential Commands |
| 03 | Text processing and Git — [notes](notes/03-text-git.md) · [quiz](quizzes/03-text-git.md) · [lab](labs/03-text-git.md) | 6 | Essential Commands |
| 04 | Users, groups and permissions — [notes](notes/04-users-permissions.md) · [quiz](quizzes/04-users-permissions.md) · [lab](labs/04-users-permissions.md) | 8 | Users and Groups |
| 05 | Processes and performance — [notes](notes/05-processes.md) · [quiz](quizzes/05-processes.md) · [lab](labs/05-processes.md) | 6 | Operations · Essential |
| 06 | Services, scheduling and logs — [notes](notes/06-services.md) · [quiz](quizzes/06-services.md) · [lab](labs/06-services.md) | 7 | Operations · Essential |
| 07 | Software and packages — [notes](notes/07-packages.md) · [quiz](quizzes/07-packages.md) · [lab](labs/07-packages.md) | 4 | Operations |
| 08 | Disks, filesystems, LVM and swap — [notes](notes/08-storage.md) · [quiz](quizzes/08-storage.md) · [lab](labs/08-storage.md) | 8 | Storage |
| 09 | Network storage and automounting — [notes](notes/09-network-storage.md) · [quiz](quizzes/09-network-storage.md) · [lab](labs/09-network-storage.md) | 5 | Storage |
| 10 | Networking — [notes](notes/10-networking.md) · [quiz](quizzes/10-networking.md) · [lab](labs/10-networking.md) | 9 | Networking |
| 11 | Network services and certificates — [notes](notes/11-network-services.md) · [quiz](quizzes/11-network-services.md) · [lab](labs/11-network-services.md) | 8 | Networking · Essential |
| 12 | Kernel, boot and recovery — [notes](notes/12-kernel-boot.md) · [quiz](quizzes/12-kernel-boot.md) · [lab](labs/12-kernel-boot.md) | 5 | Operations |
| 13 | SELinux — [notes](notes/13-selinux.md) · [quiz](quizzes/13-selinux.md) · [lab](labs/13-selinux.md) | 4 | Operations |
| 14 | Virtual machines and containers — [notes](notes/14-vms-containers.md) · [quiz](quizzes/14-vms-containers.md) · [lab](labs/14-vms-containers.md) | 5 | Operations |
| A | Appendix: Bash scripting — [notes](notes/A-scripting.md) · [lab](labs/A-scripting.md) | 6 | supporting skill |
| E | **Exam preparation** — [strategy](exam/strategy.md) · [mock exam 1](exam/mock-exam-1.md) · [mock exam 2](exam/mock-exam-2.md) | 6 | all |

Do 01–06 in order; they are the toolkit for everything else. After that, follow the
order or jump to your weakest domain. Sit both mock exams **under exam conditions**:
two hours, a timer, `man` only.

---

## Every official competency, and where it is taught

Tick the box when you can do it from memory, on a clean machine, without notes.

### Operations Deployment — 25%

| Competency | Chapter | ✓ |
|---|---|---|
| Configure kernel parameters, persistent and non-persistent | 12 | ☐ |
| Diagnose, identify, manage, and troubleshoot processes and services | 05, 06 | ☐ |
| Manage or schedule jobs for executing commands | 06 | ☐ |
| Search for, install, validate, and maintain software packages or repositories | 07 | ☐ |
| Recover from hardware, operating system, or filesystem failures | 08, 12 | ☐ |
| Manage Virtual Machines (libvirt) | 14 | ☐ |
| Configure container engines, create and manage containers | 14 | ☐ |
| Create and enforce MAC using SELinux | 13 | ☐ |

### Networking — 25%

| Competency | Chapter | ✓ |
|---|---|---|
| Configure IPv4 and IPv6 networking and hostname resolution | 10 | ☐ |
| Set and synchronize system time using time servers | 10 | ☐ |
| Monitor and troubleshoot networking | 10 | ☐ |
| Configure the OpenSSH server and client | 11 | ☐ |
| Configure packet filtering, port redirection, and NAT | 11 | ☐ |
| Configure static routing | 10 | ☐ |
| Configure bridge and bonding devices | 10 | ☐ |
| Implement reverse proxies and load balancers | 11 | ☐ |

### Storage — 20%

| Competency | Chapter | ✓ |
|---|---|---|
| Configure and manage LVM storage | 08 | ☐ |
| Manage and configure the virtual file system | 09 | ☐ |
| Create, manage, and troubleshoot filesystems | 08 | ☐ |
| Use remote filesystems and network block devices | 09 | ☐ |
| Configure and manage swap space | 08 | ☐ |
| Configure filesystem automounters | 09 | ☐ |
| Monitor storage performance | 09 | ☐ |

### Essential Commands — 20%

| Competency | Chapter | ✓ |
|---|---|---|
| Basic Git operations | 03 | ☐ |
| Create, configure, and troubleshoot services | 06 | ☐ |
| Monitor and troubleshoot system performance and services | 05, 06 | ☐ |
| Determine application and service specific constraints | 05, 06 | ☐ |
| Troubleshoot diskspace issues | 02 | ☐ |
| Work with SSL certificates | 11 | ☐ |

### Users and Groups — 10%

| Competency | Chapter | ✓ |
|---|---|---|
| Create and manage local user and group accounts | 04 | ☐ |
| Manage personal and system-wide environment profiles | 04 | ☐ |
| Configure user resource limits | 04 | ☐ |
| Configure and manage ACLs | 04 | ☐ |
| Configure the system to use LDAP user and group accounts | 04 | ☐ |

---

## How each chapter works

```text
lfcs-course/
├── notes/NN-topic.md        the teaching — concepts, worked examples, gotchas
├── quizzes/NN-topic.md      check your understanding before the lab
├── labs/NN-topic.md         hands-on tasks, each with a Verify step
├── solutions/NN-topic.md    lab solutions and quiz answers, with reasoning
├── labs/lab-environment.md  the Vagrant lab every chapter uses
└── exam/                    strategy and two timed mock exams
```

| Step | File | Why |
|---|---|---|
| 1 | **Notes** | Read with a terminal open; run every example |
| 2 | **Quiz** | Under 80%? Reread before the lab |
| 3 | **Lab** | 🟢 warm-up → 🔵 practical → 🔴 challenge; every task has a **Verify** |
| 4 | **Solutions** | Only after you have an answer you believe, even a wrong one |

### Why Verify matters more for this exam

LFCS is marked by scripts that inspect your machines afterwards. A service that is
running but not enabled, a mount that works now but is not in `/etc/fstab`, a firewall
rule that is live but not persistent — all score zero. The Verify step builds the habit
of checking **persistence** and **state**, which is where most candidates lose marks.

---

## Lab setup

[`labs/lab-environment.md`](labs/lab-environment.md) builds the lab with Vagrant: two
Ubuntu nodes (matching the exam) and one Rocky Linux node for SELinux, each with spare
disks.

---

## How to study

1. **Type the commands.** Pasting teaches nothing.
2. **Use `man`, never the browser** — from day one. In the exam it is all you have.
3. **Break things on purpose.** Every chapter has a Gotchas section; cause each failure,
   then recover.
4. **Do the 🔴 tasks.** They are closest to real exam tasks.
5. **Snapshot before each lab.** `vagrant snapshot restore <vm> clean` undoes anything.
6. **Time yourself from chapter 08 onwards.** The exam allows about six minutes per task.

---

## Progress

| Chapter | Notes | Quiz | Lab |
|---|---|---|---|
| 01 Introduction and the shell | ☐ | ☐ | ☐ |
| 02 The file system | ☐ | ☐ | ☐ |
| 03 Text processing and Git | ☐ | ☐ | ☐ |
| 04 Users, groups and permissions | ☐ | ☐ | ☐ |
| 05 Processes and performance | ☐ | ☐ | ☐ |
| 06 Services, scheduling and logs | ☐ | ☐ | ☐ |
| 07 Software and packages | ☐ | ☐ | ☐ |
| 08 Disks, filesystems, LVM and swap | ☐ | ☐ | ☐ |
| 09 Network storage and automounting | ☐ | ☐ | ☐ |
| 10 Networking | ☐ | ☐ | ☐ |
| 11 Network services and certificates | ☐ | ☐ | ☐ |
| 12 Kernel, boot and recovery | ☐ | ☐ | ☐ |
| 13 SELinux | ☐ | ☐ | ☐ |
| 14 Virtual machines and containers | ☐ | ☐ | ☐ |
| A Bash scripting | ☐ | | ☐ |
| Mock exam 1 — score: ___ / 100 | | | ☐ |
| Mock exam 2 — score: ___ / 100 | | | ☐ |

---

## License

© 2026 Costel Iacob. This course is licensed under the
[Creative Commons Attribution 4.0 International License (CC BY 4.0)](LICENSE).

You may share and adapt it for any purpose, including commercially, as long as you give
credit — for example: *"Based on LFCS Preparation by Costel Iacob,
https://github.com/Costel03/lfcs-course, CC BY 4.0"* — and indicate what you changed.

LFCS is a certification of the Linux Foundation. This course is independent and not
endorsed by or affiliated with the Linux Foundation.
