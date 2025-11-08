+++
title = "Backup Strategy"
date = 2025-11-06
+++


## ✍️ Proposed Blog Article Structure

### 1. Introduction: Why the Switch? (Duplicity vs. Restic)

* **Reference to Low Complexity Self Hosting:** Brief look back at the previous article and the former backup strategy (**Duplicity** + **Superdüp**).
* **Original Choice Duplicity:** **Open Source** with the option for **commercial support**.
* **The Decisive Disadvantage:** Duplicity does **not offer Append-Only Backups**.
* **The New Choice Restic:** Supports **Append-Only Backups** (especially with Backblaze).
* **Restic Overview:** **Large community**, **large ecosystem**, but **no commercial support**.
* **The Essential Protection:** The **Append-Only mode** of the backup provides the **crucial protection** against ransomware.

---

### 2. Requirements Profile for the New Orchestration

* **Problem Statement:** No popular solution was found that meets the requirements profile.
* **Detailed Requirements:**
    * **Sequential execution** of multiple backup steps.
    * **Simple configuration** of backup jobs.
    * **Automatic sending** of summary emails after each sequence.
    * **Heathchecks.io Ping** for monitoring.
    * Function to **wait for an internet connection**.
    * **Monitoring of backups** for irregularities (**Ransomware Junk Backup Attack**).
    * **Storing credentials** outside of the system to be backed up.

---

### 3. Evaluation and Discarding of Alternatives

* **Evaluated Alternatives:** **Backrest**, **Restic Profiles**, **Autorestic**.
* **Reason for Discarding Backrest:** Runs as a **server** (port open to the outside), keeps **credentials in memory**—represents a direct **attack vector**.
* **Reasons for Discarding Restic Profiles / Autorestic:**
    * **No possibility for sequential execution**.
    * **No pre-built email summary functionality**.

---

### 4. The New Custom Development (Architecture)

* **Decision:** Develop a custom solution, as no existing solution meets the profile.
* **Environment Constraint:** **Exclusively Shell and Python** on the target system.
* **Change in Approach:** **Programming in Go is fun, but not in YAML ("Gammel")**. Switch to a **scripting language** for flow control.
* **Re-Engineering:** The **Go executable** from the initial (discarded) approach was **re-engineered (vibe engineered)** to handle complex tasks.
* **Final Orchestration Structure:**
    * **Flow Control:** **Very simple Bash script**.
    * **Complexity Offloaded (Go Executable):** Responsible for **status report**, **junk-attack analysis**, email sending, health checks, waiting for online status.

---

### 5. Security Concept and Obscurity

* **Infrastructure for Security Hardening:**
    * Script and Go executable run on a **separate system** (**Raspberry Pi Zero**).
    * **Raspberry Pi Zero Hardening:**
        * Accessible **only from Notebook** via **password-protected SSH key**.
        * **No other services** are running, only the **Cronjob** starts the backup script.
* **Backup Preparation:** The separate system connects to the host via **SSH**, copies a **fresh Restic executable**, and passes **passwords via environment variable**.
* **Advantage of Offloading:** The **detection of Junk Backup Attacks** runs **outside** of the secured system.
* **Open Vulnerability (Limits of Security):** Does **not offer 100% protection** (root attacker can harvest passwords).
* **Gained Advantage (Obscurity):** An attacker finds **no hints about the existence of a backup** on the host.
* **Essential Protection:** The **Append-Only mode** of the backup provides the **crucial protection** against data deletion.