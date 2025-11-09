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

It seems like, despite restics large community, none of the existing solution meets my requirements, so i decided to once again, run my own. I first contemplated building a suite of restic helpers for email-notification, summary generation etc. that i could integrate with resticprofiles but soon realized that with this approach i would end up programming in YAML, wich is awful. Instead, i wanted a simple language that has great support for handling environment variables, serial execution of external applications and handling their outputs and exit codes. So in short: good old Bash! 


The setup i finally settled on is published on github as [restic-kit](https://github.com/lackhove/restic-kit). At its core, restic-kit is configured via a dead simple shell script that comprises multiple restic calls like this
```bash
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


## Security and Obscurity

Going back to my initial list of requoremtns, the combnation of restic-kit and a simple sysmted timer supports  Sequential execution, Simple configuration, Automatic summary email sending, backup monitoring and Heathchecks.io Ping. Whats still missing is the ransomare-proofness, that initially triggered this whole endaevour. Benjamin Ritter [first described](https://medium.com/@benjamin.ritter/how-to-do-ransomware-resistant-backups-properly-with-restic-and-backblaze-b2-e649e676b7fa) how to set up restic with backblaze for WORM backups, so i am going to reiterate this here. I only deviated from his setup by setting `daysFromHidingToDeleting` to  `null`, so nothing will get automatically deleted. Instead, i set an appointment in my calendar to manually check the backups integrity and delete the hidden files manually every few months.

Even with the WORM strategy being the central protection against ransomware attacks, i still prefer not storing the backup credentials and performing the irregularity scanning and notifying on the server itself. This contrasts the requirement for automated, unattaneded backups, which requires access to all credentials. The way out of this dilemma is an external backup system, which pulls the data from the server. This however requires a high bandwith network connection and sufficient CPU power to perform chunking, deduplication and encryption. For ma home serer setup, the invest and and operational costs of such a sysstem seems overkill, so instead i built the next best solution baed on a raspberry pi zero i still had in my hobby drawer.

The setup is as follows: The Pi runs a minimal read-only OS that runs nothing but the backup  and is only accessible via a password protected ssh key. The backup shell script is triggered by a systemd timer and executes most restic commands and all restic-kit commands locally. Only the restic backup command is executed on the server via ssh with credeitals being passed as environment variables using the SSHs SetEnv functionality. For each bakup run, a clean restic executable is copied to the server and deleted afterwards. This is designed to prevent an attacker from tampering with the restic executable and more importantly, to not leave any trace of the backup on the server itself. Even with root access, an attacker wont find a single hint in ther servers file system that there is a backup to begin with, making it more unlikely for my backup being targeted in a ransomware attack. In addition, all checks and notifications run on an isolated system, so even in case of an attack, i can still hope for my monitopring to notice.

 This is however just **obscurity** and shouldnt be mistaken as **security**. An attacker with root access and sufficient knowledge will still be able to eavesdrop on the backups credentials and empty my snapshots. The only security here is provided by the WORM configuration and occasional backups to an external hard drive stored offsite.


## Future Work and Final Thoughts

The system now has been running for a few weeks and so far, works as expected. When it comes to ransomeare proofness, i hope that i will never have the chance to verify its effectiveness, but at least the known  vulurabilities are mitigated. Ther is always room for improvement, so in the coming weeks, i might extend the `audit` functionality to scan for suspicious snapshots patterns and auto update the restic executables.

The whole project ended up being much larger in terms of LOC than superdup, partly because golang is a more verbose language, but mostly because it just has more features. A year ago, i would have probably settled on a simpler solution, but with the advent of LLM coding agents, the implementation only took a few hours. The whole project was vibe-engineered, with just a handful of lines written with my bare hands but every single line carefully reviewed. The resulting code is not as good as if it was hand-crafted, but good enough for a project a user base of probaly just me. This highlights an interesting aspect:  Instead of contributing to an existing project on github i could just build my own in a matteer of hours for a few dollars woth of tokens. Building bespoke software has become cheap. I am not sure what this means for the FOSS ecosystem, but i want to be optimistic.