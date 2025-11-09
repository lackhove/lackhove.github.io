+++
title = "Backup Strategy"
date = 2025-11-06
+++

When i was a teeenager and still living at my parents, a lightning struck our neighborhood, basting sockets from the walls and frying most electronic devices in our home, including the tv, telephones and most importantly my PC. Luckily, the most important data on its harddisk were a couple of savegames and my handcrafted Debian (potatohead?) installation. Nonetheless, this taught my an important lesson about backups early on and ive been very serious about backups ever since.

In already mentioned my backup strategy in my last  article about my low complexity self hosting stack. The strategy involved my [automation](https://gitlab.com/lackhove/superdup) and [duplicacy](https://github.com/gilbertchen/duplicacy). What i particuarly like about duplicacy is that its *paid* open souce software. While it offers all the benfits of FOSS, someone builing a business around it make sthe project much more sustainable and gives me the option to buy support in case of a catastrophic error.

however, While contemplating if this would make for a good follw up article, i realized my setup had onehuge caveat: It was not ransomware-proof: The backup ran unattended, from the same machine, so i had no choice but storing the credentials for the storage on the device itself. This allows for An attacker to just use these secrets to delete my backup before encrypting my data. The go to solution in this situation is usually WORM (Write Once Read Many) or append-only backups: In this scenario, the credentials only give read and write acces, but no delete or overwrite. Unfortunately, this is not supported by duplicacy. In XXXX, ransomware wasnt a thing, but in 2025 this ha become a majot threat. Enter my shiny new restic and restic-kit based backup strategy.

## Shopping for a backup tool

Without commercial support, the most important aspect of a duplicacy successor is its popularity. restic has XXX stas on github and a thriving ecosystem, so i dint bother looging further. Most importantly, restics S3 backend doesnt need delet epermissions, allowing for a simple WORM implementation in conjuction with Backblaze, as decreibed by XXX.

In summary,  my requirements for a backup solution are:
* **FOSS** for philosphcal reasons
* **Good project health** so that i can run this setup for the next 10 years
* **Encrypted, incremental, chunk based backups to S3** just like duplicacy
* **Sequential execution** of multiple backup steps, i.e backup multiple paths, forget, prune and check in one go.
* **Simple configuration** i can stil remember six years later when my server just broke and i need to fix it ASAP.
* **Automatic sending** of summary emails after each sequence, so i can check on the backup state wih´thout running a web server.
* **Heathchecks.io Ping** for monitoring so i get notified when the backup stops working
<!-- * **Handling of missing internet connection** so the backup still works wehen my DSL connetion resets. -->
* **Monitoring of backups** for irregularities so i can notice Ransomware Junk Backup Attacks whe an attacker would create broken snapshots and wait wor the good ones to be deleted by the prune policy before they encrypt the target system.
* **Storing credentials** outside of the system to be backed up as an additional ransomware protection besides WORM.

While restic ticks the first three boxes, the remaining requirements are out of its project scope. May orcheatrations tools exist for restic with the tree most poplar ones being

### Backrest
Backrest is a web GUI for restic and seems to be th most popular solution (on github). While it looks enticing, i am not cofortable with running a server for my backups that opens a port and hence creates an additional attack vector. Besides, storing the credentials outside the system will be difficult with this design.

### resticprofiles and autorestic

The both have a very similar approach and basically allow for setting most / all restic parameters in a configuration file. The individual actions such as backup, check, forget are triggered via systemd timers or cron. resticprofiles supports heathchecks.io pings, but neither soution can generate (and send) email notifications or monitor for irregularities. What really grind my gears s however, that they both cannot run all the backup steps sequentially. You have to create timers for each step, so e.g. run your backups at 01:00, forget at 02:00 and prune at 3:00, which wil break havoc when one of these steps takes longer.


## Building my own orchestration, with blackjack and golang

It seems like, despite restics large community, none of the existing solution meets my requirements, so i decided to once again, run my own. I first contemplated building a suite of restic helpers for email-notification, summary generation etc. that i could integrate with resticprofiles but soon realized that with this approach i would end up programming in YAML, wich is awful. Instead, i wanted a simple language that has great support for handling environment variables, serial execution of external applications and handling their outputs and exit codes. This sounds just like the vererable Bash! 


The setup i finally settled on is published on github as [restic-kit](https://github.com/lackhove/restic-kit). At its core, restic-kit is configured via a dead simple shell script that comprises multiple restic calls like this
```bash
echo "Getting repository snapshots..."
$RESTIC snapshots \
    --group-by=paths \
    --json > "$TEMP_DIR/snapshots.out" 2> "$TEMP_DIR/snapshots.err"
echo $? > "$TEMP_DIR/snapshots.exitcode"
```
Here, the restic "snapshots" action is executed and its STDOUT, STDERR and exit code stored i dedicated files inside a temporary directory. All secrets are passed as environment variables so they dont appear in the process list. At the end of the script a simple golang executable parses these files for each step.

These more complex tasks are implemented in a single `restic-kit` golang executable, e.g.
```bash
$RESTIC_KIT notify-http \
    --url "https://hc-ping.com/<uuid>" \
    "$TEMP_DIR" || true
```
which parses all files in the temporary directory and pings e.h. healthchecks.io with success or fail, depeneding on the exit codes.

The `restic-kit` executable has several subcommands:
* **notify-email**: Send an email notification with overall success and a short summary for each restic command:
    ```
    Overall Status: SUCCESS

    ✅ backup etc
    Files: 0 new, 0 changed, 300 unmodified
    Directories: 0 new, 0 changed, 159 unmodified
    Data added: 0 B (0 B packed)
    Total files processed: 300
    Total bytes processed: 23.4 MB
    Duration: 3.64 seconds

    ✅ forget
    4 snapshots removed

    ✅ check
    PASSED

    ✅ snapshots
    Repository Snapshots: 27

    Path: /etc
    Snapshots: 9
    Date & Time      | New | Modified | Total Files | Added Size | Total Size
    ---------------- | --- | -------- | ----------- | ---------- | ----------
    2025-11-08 14:12 |   0 |        0 |         300 |        0 B |    23.4 MB
    2025-11-07 02:30 |   0 |        1 |         299 |    57.5 KB |    23.4 MB
    2025-11-06 02:30 |   0 |        1 |         299 |    57.5 KB |    23.4 MB
    2025-11-05 02:30 |   0 |       12 |         299 |     5.6 MB |    23.4 MB
    2025-11-04 02:30 |   0 |        1 |         299 |    57.5 KB |    23.4 MB
    2025-11-03 02:30 |   0 |        1 |         299 |    57.5 KB |    23.4 MB
    2025-11-02 02:30 |   0 |        0 |         299 |        0 B |    23.4 MB
    2025-10-31 14:36 |   0 |        0 |         299 |        0 B |    23.4 MB
    2025-10-30 23:34 |   0 |        0 |         298 |        0 B |    23.4 MB
    ```
* **notify-http**: Perform a single HTTP GET request to notify an external service. In case an error was detected, the path `/fail` is appended, so that the correct state is displayed by e.h. healthchecks.io.
* **wait-online**: Wait for network connectivity by checking if a URL is reachable with exponential backoff.
* **audit**: Audit restic snapshots for size anomalies. Checks for unusual size changes between the two most recent snapshots per path. Sends email notifications for any failures.
* **cleanup**: Remove the log directory if all backup operations were successful. Keep it for debugging if any operations failed.




golang  produces self contained executables without any OS dependencies, which i can run directly on the server without any containers or external dependencies. This greatly simplifies setup and remote credential storage while still being relatively low-complexity.





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