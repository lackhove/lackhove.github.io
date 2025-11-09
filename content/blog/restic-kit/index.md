+++
title = "Restic-kit: Orchestrating and Monitoring a Ransomware-Proof Backup Stack"
date = 2025-11-09
+++

When I was a teenager and still living at my parents', lightning struck our neighborhood, blasting sockets from the walls and frying most electronic devices in our home, including the TV, telephones and most importantly, my PC. Luckily, the most important data on its hard disk were a couple of save games and my handcrafted Debian Sarge installation. Nonetheless, this taught me an important lesson about backups early on and I've been very serious about data preservation ever since.

I already mentioned my backup strategy in my last article about my low complexity self-hosting stack. The strategy involved my self-written orchestration tool [superdup](https://gitlab.com/lackhove/superdup) and [duplicacy](https://github.com/gilbertchen/duplicacy). What I particularly liked about duplicacy is that it is *paid* open-source software. While it offers all the benefits of FOSS, someone building a business around it makes the project (hopefully) much more sustainable and gives me the option to buy support in case of a catastrophic error.

However, while contemplating if this would make for a good follow-up article, I realized my setup had one huge caveat: It was not ransomware-proof. The backup ran unattended, from the same machine, so I had no choice but to store the credentials for the remote storage on the device itself. This allows an attacker to simply use these secrets to delete my backup before encrypting my data. The go-to solution in this situation is usually **WORM** (Write Once Read Many) or **append-only** backups. In this scenario, the credentials only grant read and write access, but no delete or overwrite permissions. Unfortunately,  [this is not supported by duplicacy](https://github.com/gilbertchen/duplicacy/issues/528). In 2020, ransomware wasn't the threat it is today, but in 2025 this has become a major concern. So it was time for an update. Enter my shiny new [restic](https://restic.readthedocs.io/en/stable/) and [restic-kit](https://github.com/lackhove/restic-kit) based backup strategy.

## Shopping for a Backup Tool

Over the years, my requirements for a backup solution have become:
* **FOSS** for philosophical reasons.
* **Good project health** so that I can run this setup for the next 10 years.
* **Encrypted, incremental, chunk-based backups to S3** so no third party can access my personal data, I don't have to upload lots of data more than once and I can freely choose the storage provider.
* **Sequential execution** of multiple backup steps (i.e. backup multiple paths, forget, prune and check in one go).
* **Simple configuration** I can still remember six years later when my server just broke and I need to fix it ASAP.
* **Automatic sending** of summary emails after each sequence, so I can check on the backup state without running a web server.
* **healthchecks.io Ping** for monitoring so I get notified when the backup stops working.
* **Monitoring of backups** for irregularities, so I can notice Ransomware Junk Backup Attacks where an attacker would create broken snapshots and wait for the good ones to be deleted by the prune policy before encrypting the target system.
* **Storing credentials** outside of the system to be backed up as an additional ransomware protection besides WORM.

Without commercial support, the most important aspect of a duplicacy successor is its popularity. [restic](https://restic.readthedocs.io/en/stable/) has over 30k stars on GitHub and a thriving ecosystem, so I didn't bother looking further. Most importantly, restic's S3 backend doesn't require delete permissions, allowing for a simple WORM implementation in conjunction with Backblaze. While restic ticks the first three boxes, the remaining requirements are out of its project scope. Many orchestration tools exist for restic, with the three most popular ones being [backrest](https://github.com/garethgeorge/backrest), [resticprofile](https://creativeprojects.github.io/resticprofile/) and [autorestic](https://autorestic.vercel.app/):

### Backrest
Backrest is a web GUI for restic and seems to be the most popular solution (on GitHub). While it looks enticing, I am not comfortable with running a server for my backups that opens a port and hence creates an additional attack vector. Besides, storing the credentials outside the system would be difficult with this design.

### Resticprofile and Autorestic

Resticprofile and autorestic both have a very similar approach and basically allow for setting most or all restic parameters in a configuration file. The individual actions such as `backup`, `check` and `forget` are triggered via systemd timers or cron. resticprofile supports healthchecks.io pings, but neither solution can generate (and send) email notifications or monitor for irregularities. What really grinds my gears, however, is that they both cannot run all backup steps sequentially. You have to create timers for each step, so, for example, running your backups at 01:00, forget at 02:00 and prune at 3:00, which will break havoc when one of these steps takes longer than expected.

## Building my own Orchestration, with Blackjack and Golang

It seems like, despite restic's large community, none of the existing solutions meet my requirements, so I decided to once again, roll my own. I first contemplated building a suite of restic helpers for email notification, summary generation, etc., that I could integrate with resticprofiles but soon realized that with this approach I would end up programming in YAML, which is a royal PITA. Instead, I wanted to describe the backup flow in a simple language that has great support for serial execution of external applications and handling their outputs and exit codes. So in short: good old **Bash**!

The setup I finally settled on is published on GitHub as **[restic-kit](https://github.com/lackhove/restic-kit)**. At its core, restic-kit is configured via a dead simple shell script that comprises multiple restic calls like:
```bash
$RESTIC snapshots \
    --group-by=paths \
    --json > "$TEMP_DIR/snapshots.out" 2> "$TEMP_DIR/snapshots.err"
echo $? > "$TEMP_DIR/snapshots.exitcode"
```
Here, the restic `snapshots` action is executed and its STDOUT, STDERR and exit code are stored in dedicated files inside a temporary directory. All secrets are passed as environment variables so they don't appear in the process list. At the end of the script, a single executable parses these files for each step.

These more complex tasks are implemented in the `restic-kit` Go executable, e.g.:
```bash
$RESTIC_KIT notify-http \
    --url "https://hc-ping.com/<uuid>" \
    "$TEMP_DIR" || true
```
which notifies healthchecks.io. The `restic-kit` executable has several subcommands:
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
* **notify-http**: Perform a single HTTP GET request to notify an external service (e.g., healthchecks.io)  with success or fail, depending on the exit codes parsed from the temporary directory.
* **wait-online**: Wait for network connectivity by checking if a URL is reachable with exponential backoff.
* **audit**: Audit restic snapshots for size anomalies. Checks for unusual size changes between the two most recent snapshots per path. Sends email notifications for any failures.
* **cleanup**: Remove the log directory if all backup operations were successful. Keep it for debugging if any operations failed.

Go produces self-contained executables without any OS dependencies, which I can run directly on the server without any containers or external dependencies. This greatly simplifies setup and remote credential storage while still being relatively low-complexity.

## Security and Obscurity

Going back to my initial list of requirements, the combination of restic-kit and a simple systemd timer supports sequential execution, simple configuration, automatic summary email sending, backup monitoring and healthchecks.io Ping. What's still missing is the ransomware-proofness that initially triggered this whole endeavor. Benjamin Ritter [first described](https://medium.com/@benjamin.ritter/how-to-do-ransomware-resistant-backups-properly-with-restic-and-backblaze-b2-e649e676b7fa) how to set up restic with Backblaze for WORM backups, so I am going to reiterate this here. I only deviated from his setup by setting `daysFromHidingToDeleting` to `null`, so nothing will get automatically deleted. Instead, I set an appointment in my calendar to manually check the backup's integrity and delete the hidden files manually every few months.

Even with the WORM strategy being the central protection against ransomware attacks, I still prefer not storing the backup credentials and performing the irregularity scanning and notifying on the server itself. This contrasts the requirement for automated, unattended backups, which requires access to all credentials. The way out of this dilemma is an external backup system, which pulls the data from the server. This however requires a second server to perform chunking, deduplication and encryption. For my selfhosting setup, the investment and operational costs of such a system seem overkill, so instead I built the next best solution based on a Raspberry Pi Zero I still had in my hobby drawer.

The setup is as follows: The Pi runs a minimal read-only OS that runs nothing but the backup and is only accessible via a password-protected SSH key. The backup shell script is triggered by a systemd timer and executes most restic commands and all restic-kit commands locally. Crucially, the `restic-kit audit` command runs completely outside of the server on this secured system. Only the restic backup command is executed on the server via SSH, with credentials being passed as environment variables using the SSH `SetEnv` functionality. For each backup run, a clean restic executable is copied to the server and deleted afterward. This is designed to prevent an attacker from tampering with the restic executable and, more importantly, to not leave any trace of the backup on the server itself. Even with root access, an attacker won't find a single hint in the server's file system that there is a backup to begin with, making it less likely for my backup to be targeted in a ransomware attack.

This is, however, just **obscurity** and shouldn't be mistaken as **security**. An attacker with root access and sufficient knowledge will still be able to eavesdrop on the backup's credentials and empty my snapshots. The only true security here is provided by the WORM configuration and occasional backups to an external hard drive stored offsite.

## Future Work and Final Thoughts

The system has now been running for a few weeks and so far, works as expected. When it comes to ransomware-proofness, I hope that I will never have the chance to verify its effectiveness, but at least the known vulnerabilities are mitigated. There is always room for improvement, so in the coming weeks, I might extend the `audit` functionality to scan for suspicious snapshot patterns and auto-update the restic executables.

The whole project ended up being much larger in terms of LOC than superdup, partly because Go is a more verbose language, but mostly because it just has more features. A year ago, I would have probably settled on a simpler solution, but with the advent of LLM coding agents, the implementation only took a few hours. The whole project was **[vibe-engineered](https://simonwillison.net/2025/Oct/7/vibe-engineering/)**, with just a handful of lines written with my bare hands but every single line carefully reviewed and the overall software designed myself. The resulting code is not as good as if it was hand-crafted, but good enough for a project with a user base of probably just me. This highlights an interesting aspect: Instead of contributing to an existing project on GitHub, I could just build my own in a matter of hours for a few dollars worth of tokens. Building bespoke software has become cheap. I am not sure what this means for the FOSS ecosystem, but I want to be optimistic.