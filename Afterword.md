# Afterword

**Author:** Usman O. Olanrewaju (Blu3 Sky)

**Date:** 2026/07/01

**Focus:** Exam Experience, Protocol, and Guidance for Future Candidates

---

## Table of Contents

1. [Certification Result](#1-certification-result)
2. [Things to Know](#2-things-to-know)
3. [Contact](#3-contact)

---


## 1. Certification Result


**Credential:** [Red Hat Certified System Administrator (RHCSA)](https://www.credly.com/badges/b63664f0-67d2-455d-99b5-5858cb497056/public_url)

**Exam:** EX200 on Red Hat Enterprise Linux 10

**Score:** 237 / 300 — PASS (threshold: 210)

**Date:** 2026/06/29

**Valid Until:** 2029/06/29

### Objective Breakdown

| Objective                              | Score |
|----------------------------------------|-------|
| Understand and use essential tools     | 64%   |
| Manage software                        | 50%   |
| Create simple shell scripts            | 0%    |    
| Operate running systems                | 75%   |
| Configure local storage                | 100%  |
| Create and configure file systems      | 100%  |
| Deploy, configure and maintain systems | 89%   |
| Create and configure file systems      | 100%  |
| Manage basic networking                | 100%  |
| Manage users and groups                | 100%  |
| Manage security                        | 100%  |

### Honest Notes on the Result

Three hours for 21 tasks is tight. My shell scripting task zeroed out — not because I don't know how to write a script, but because time ran out. The exam included NFS, two AutoFS mounts, Flatpak, and a systemd daemon with both a `.service` and `.timer` unit. Storage, networking, security, and user management are all learnable to 100% with deliberate lab practice. Manage your time ruthlessly, do not get stuck on a single task; If a task is going to get your time and you have 5 more, move and comeback, much better to lose a mark than 5 marks



## 2. Things to Know

### 2.1 Documentation in This Repo

The documentation here is a combination of my own hands-on knowledge and three core references:

- **William Shotts** — *The Linux Command Line* (the book that started it all)

- **Asghar Ghori** — *RHCSA Red Hat Enterprise Linux 10* (primary exam reference)

- A third Linux book to be added — details coming later

> You need to practice, practice, and practice. Reading alone will not pass this exam. The lab environment is brutal — but it is honest. Think beyond the textbook and build intuition on the terminal.


### 2.2 Before the Exam

**Voucher:** The exam voucher retails at approximately $500 USD. I obtained mine at a discount through a partner company. Research local Red Hat partners — discounts do exist.

**Practice:** Get your hands dirty on the terminal every single day. The goal is not the certificate — the certificate is a test of what your brain already knows. Build the brain first.

**Typing speed:** My typing speed at exam time was approximately 67 WPM. Higher is better — every second counts under time pressure. Practice accordingly.

**Compatibility test:** Run the Red Hat compatibility check on your exam hardware before the exam day. Confirm webcam, ethernet, and display all pass. Do not leave this for the last minute.

**Room preparation:** Clear your desk completely. The proctor performs a 360-degree room scan including the ceiling. Have nothing within reach — no books, no phones, no notes.

### 2.3 During the Exam

- You will have **three terminals** available, not two:
  - One terminal on the main interface — do not configure anything here.
  - Two console terminals via the exam environment — these are your working nodes.

- To access the exam environment: click `start console`, then click `console vm` to bring up the node.

- The console terminals can also be used for `man` page lookups.

- After completing each task, always ask yourself: "Does this service need to be enabled or started?" If yes, enable it — or install it from the DNF repo you configured.

- Time conversion: if a task specifies "2 PM", that is 14 in cron — always use 24-hour format in crontab entries.

- Command paths: if the task specifies `/bin/echo`, use `/bin/echo` exactly — do not substitute `echo` even though they behave the same. Use what the task provides.

- Books and phones are **not permitted** anywhere near your workspace.

- Do not speak during the exam.

### 2.4 Proctor Protocol (PSI)

- **Face the screen at all times.** Looking away — even briefly — can result in immediate exam termination.

- The proctor monitors your webcam feed, microphone audio, mouse movement, keyboard input, and screen in real time.

- Wired peripherals (keyboard, mouse) are permitted and are still tracked.

- Any behaviour the proctor considers suspicious can result in instant cancellation with no appeal.

- The proctor will introduce themselves at the start and may communicate via chat during the session.


### 2.5 After the Exam

- The exam ends the moment the 3-hour timer expires — regardless of what you are in the middle of doing. There is no grace period.

- Results are delivered via email typically within one hour of completion.

- The proctor will confirm the session end and ask if you have any questions before closing the connection.

### 2.6 Breaks

Breaks were not something I explored during my attempt — I did not request one. If you need a break, you would need to ask the proctor directly. Cannot confirm whether it is permitted or what the procedure is.



## 3. Contact

For questions — personal, exam-related, or if you want to know which practice tasks I used during preparation — reach out:

**Email:** rhcsa10.protozoan328@passinbox.com

---

> **Go further in lightness.**
                                                                    
