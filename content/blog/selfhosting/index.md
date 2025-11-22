+++
title = "The Low-Complexity Stack: Self-Hosting with OpenSUSE MicroOS and Podman"
date = 2025-10-24
+++

When the Raspberry Pi launched in 2012, I was lucky enough to snatch one of the first batches. This is when I caught the self-hosting bug. At that time, I was a student with ample spare time and few responsibilities. I viewed self-hosting as a way to reduce my dependency on the cloud (a.k.a. other people's computers) and take back ownership of my data, but most importantly as a technology playground. Thirteen years later, I have a day job and a family. The server now runs crucial applications like home automation and provides services my extended family depends on. This creates a clear conflict: The demand for high availability clashes directly with my limited spare time to maintain it. For this core server, its raison d'être has shifted entirely from a playground to an essential reliable utility. I can still keep my Raspberry Pis for fun and experimentation, but nobody but me notices when they break. Ultimately, I want to be the one to decide when I spend time on my hobby projects, not some mandatory update or software release schedule.

My ultimate solution truly boils down to just one thing: reducing complexity. This article chronicles my thirteen-year journey chasing that singular goal, detailing the traps I fell into along the way. I'll give you a tour of my current setup: A mini-ITX server with minimal moving parts, PiKVM for dedicated out-of-band management, openSUSE MicroOS providing an immutable auto-updating OS and Podman for secure systemd-integrated rootless containers.


## History

From the first Raspberry Pi on, my requirements, experience and setup evolved significantly though not always in sync. As far as I recall, the milestones were the following:

**Raspberry Pi with single 3.5" HDD and Raspbian on the SD Card**
This was basically the vanilla setup, hosting SSH and NFS servers. It was great for offloading data from my laptop’s tiny SSD, but it quickly became a lesson in maintenance hell. The system was horribly unreliable since the SD card kept failing every few weeks.

**Raspberry Pi with single 3.5" HDD and Arch Linux-ARM on a read-only SD Card**
I countered the failing SD cards by switching to Arch Linux ARM, which at that time could mount the root file system read-only. This dramatically reduced SD card wear and made the system immune to power outages. This was my first major step toward minimizing maintenance and building a reproducible system.

**Raspberry Pi with single 3.5" HDD and Arch Linux-ARM on a read-only SD Card with Docker**
When my use cases and the list of hosted services grew, I faced two mounting problems: the need to improve service isolation to reduce the attack surface and the tedious and boring chore of manually installing and setting up services and their dependencies. Docker seemed to be a way out of configuration complexity while also improving separation.

**x86 mini-ITX server with RAID 1, CentOS and Docker**
The Raspberry Pi had hit a wall: the supply of pre-built ARM container images was short and building them on the Pi was slow. More critically, the single hard drive was a significant reliability risk. The next evolution was an ASRock mini-ITX board with an Intel N3150 CPU, two 3.5" HDDs in a RAID 1 and CentOS on an SSD. This hardware finally allowed for RAID 1, making the system much more reliable. With CentOS and Docker, I had a very solid familiar OS that was a keeper for a few years.

**x86 mini-ITX server with RAID 1, CentOS and Podman**
While containers were a huge win, Docker had its issues. Running the daemon as root was sub-par from a security perspective and the lacking integration with the underlying OS made management cumbersome. Then Podman came along, solving both issues with a traditional fork/exec model, tight systemd integration and the ability to run rootless containers—containers without privileged permissions. This was a massive security and simplicity upgrade.

**x86 mini-ITX server with RAID 1, Fedora Atomic Host / Fedora CoreOS**
Despite the last setup working great, it still had two disadvantages: OS updates required manual intervention and the root file system was still writable, meaning power outages could still wreck the system. The setup still felt more like a pet than cattle. Fedora CoreOS solved these issues by performing atomic updates (updating the OS as a whole and enabling simple rollbacks) and bringing back the immutable root file system. This offered protection from power outages without the cost and added complexity of a UPS. It delivered on the promise of a secure always up-to-date and reliable system.

**x86 mini-ITX server with RAID 1, openSUSE MicroOS**
My only remaining peeve with Fedora CoreOS was the slow updates due to `ostree` and that it often broke and required manual intervention during major OS version upgrades. I realized a rolling-release distro could avoid this particular complexity chore. I was delighted to discover openSUSE MicroOS, an immutable container host system like Fedora CoreOS but with Btrfs instead of `ostree` and—tadaa—a rolling release model! Without major OS release upgrades, the system has been absolutely rock solid for me since summer 2020.


## If It's Complicated, It's a Chore

My main takeaway from 13 years of self-hosting is thatwhen moving from hosting a simple file server for yourself to a range of critical services for family and friends, the **top priority is reducing complexity**.
Every additional piece of hardware, daemon, container, service, or even configuration file increases your cognitive load and the likelihood of failures, issues and security risks. Or to put it more vividly: You really don't want to start debugging your Kubernetes setup when an update made the living room lights go haywire on Christmas Eve.

Committing to reducing complexity gives me the following essential benefits:

* **Automated Maintenance and Recovery:** openSUSE MicroOS's transactional design and Podman's auto-updating containers handle updates automatically and roll back bad ones. Furthermore, Btrfs snapshots enable instant recovery of the host OS to a known good state with a single command. In case of a disaster, the stack's inherent simplicity allows the system to be restored with a tool as simple as rsync.

* **Inherent Security by Design:** By keeping the system minimal, up-to-date and exposing only what's necessary, the attack surface is drastically minimized. If a container is breached, the impact is (hopefully) contained by SELinux, rootless Podman containers and having a minimal immutable host OS.

* **Minimized Points of Failure:** Fewer moving parts means fewer things that might break. I deliberately chose not to use layers like hypervisors, orchestration or container management GUIs. My stack only has two levels. This prevents failures and makes diagnosing any issues straightforward.

* **Reduced Cognitive Load:** I log on to my server every few months, so the KISS principle is law. Since I've avoided complex layers, my entire infrastructure is defined by simple systemd units and human-readable configuration files. Simple solutions are always easier to debug when you're under pressure.


## My Low-Complexity Self-Hosting Stack

The following breakdown details the rationale and design choices behind each level of my stack. My goal here is to give an overview of the architectural decisions and highlight pitfalls, not to offer a tutorial. That might be something for a future post.

### Level 0: The Hardware

As mentioned earlier, the hardware is a mini-ITX server with an Intel N100 low-power CPU and a cheap SSD for the OS. The most interesting part is what the system doesn't have: a fan. The only mechanical parts that will fail are the two 3.5" HDDs, which are redundant as RAID1.
The OS is installed on bare metal, so no complexity is added by a hypervisor such as Proxmox. Instead, there is a [PiKVM](https://pikvm.org/) attached to the server, which gives me full control of the hardware, independent of the OS:
* If the OS fails to boot or a hardware error occurs, I can still access the BIOS, push power and reset buttons and virtually plug in recovery ISOs from anywhere.
* The PiKVM is only accessible from within my home network (or Wireguard when not at home), so one less attack vector I need to worry about.
* The server can operate without the PiKVM, so this adds zero complexity to the stack.

### Level 1: The Operating System

The feature I like most about openSUSE MicroOS is probably that I don't have to think about it at all. The OS is immutable and updates are installed by just rebooting into a new Btrfs snapshot. When something goes wrong, it just reboots into the last working snapshot. However, this has never happened to me. MicroOS has a rolling release model, so there are no major upgrades that might break stuff. And with the very few packages it ships, chances for breakages are extremely low anyway.

Speaking of few packages: I have only added the bare minimum to the default list, such as vim and tmux. I also limited the configuration to the basic essentials such as keyboard layout, time zone, SSH, mounts, etc. and mdadm email notifications, which I haven't containerized yet.
Everything else goes into containers.

The key feature of MicroOS, reliable, automatic updates, must be enabled manually via:
```bash
$ systemctl enable --now transactional-update.timer
```
and the desired reboot interval must be set in /etc/systemd/system/transactional-update.timer.d/local.conf and method in /etc/transactional-update.conf. I opted for daily and REBOOT_METHOD=systemd, since it's the simplest.

With this minimal setup, restoring the OS is only a matter of restoring the /etc and the container data dirs and installing the additional packages. No complex orchestration tools such as Ansible, Salt, Puppet, etc., are required.

### Level 2: Podman

Thanks to Podman's first-class systemd integration, all I need to manage my container lifecycle is systemd. So again, no need for container management or orchestration systems such as Docker Compose or Kubernetes that add complexity to the stack. Instead, the whole system can be managed with a handful of simple tools, i.e., `systemctl` and `journalctl`. With the integration of Quadlet into Podman, I don't even need to manually write, manage and update systemd units anymore. Instead, all my containers are managed via simple INI files in /etc/containers/systemd. The corresponding systemd unit files are created at boot (or `systemctl --user daemon-reload`) and even the user's containers are managed right in the /etc directory at /etc/containers/systemd/users/<uid>/, which further simplifies backup and restore. A simple example would be:
```ini
# /etc/containers/systemd/users/<uid>/mosquitto.container
[Service]
Restart=on-failure
TimeoutStopSec=70

[Container]
ContainerName=mosquitto
Image=docker.io/eclipse-mosquitto:2
AutoUpdate=registry
PublishPort=1883:1883
PublishPort=9001:9001
Volume=/var/lib/docker-confs/mosquitto/config:/mosquitto/config/:Z,U
Volume=/var/lib/docker-confs/mosquitto/data:/mosquitto/data/:Z,U
Volume=/var/lib/docker-confs/mosquitto/log:/mosquitto/log/:Z,U

[Install]
WantedBy=default.target
```

Isn't that beautiful? Here I defined that, among other things, the container image should be auto-updated (see below), publish ports and mount volumes with correct SELinux labels and auto-adjusted permissions.


#### Auto-updates

As with the OS updates, automatic container updates are disabled by default. Activating them is as easy as enabling the `podman-auto-update.timer` unit. However, this will produce lots of unused images, so I created these two small units to clean up unused images / containers periodically:
```ini
# /etc/systemd/system/podman-prune.service
[Unit]
Description=Podman prune service
Wants=network.target
After=network-online.target

[Service]
ExecStart=/usr/bin/podman system prune -a --volumes -f

[Install]
WantedBy=multi-user.target default.target
```
and:
```ini
# /etc/systemd/system/podman-prune.timer
[Unit]
Description=Podman prune timer

[Timer]
OnCalendar=weekly
Persistent=true

[Install]
WantedBy=timers.target
```

#### Pods instead of Compose-Files

For multi-container applications, we can leverage Podman's pods, which basically allow network communication between containers over localhost ports, sparing me the complexity of container networks. [Paperless-ngx](https://docs.paperless-ngx.com/), for instance, requires a redis container in addition to the application itself. So for this, I have the following container files:

```ini
# paperlessngx-redis.container
[Service]
Restart=on-failure
TimeoutStopSec=70

[Container]
ContainerName=paperlessngx-redis
Pod=paperlessngx.pod
Image=docker.io/redis:8
AutoUpdate=registry
```
```ini
# paperlessngx-web.container
[Service]
Restart=on-failure
TimeoutStopSec=70

[Container]
ContainerName=paperlessngx-web
Pod=paperlessngx.pod
Image=ghcr.io/paperless-ngx/paperless-ngx:latest
AutoUpdate=registry
Volume=/var/lib/docker-confs/paperless-ngx/data:/usr/src/paperless/data:Z,U
Volume=/var/lib/docker-confs/paperless-ngx/media:/usr/src/paperless/media:Z,U
Volume=/var/lib/docker-confs/paperless-ngx/export:/usr/src/paperless/export:Z,U
Environment=PAPERLESS_REDIS=redis://localhost:6379 PAPERLESS_SECRET_KEY=<redacted> PAPERLESS_TIME_ZONE=Europe/Berlin PAPERLESS_OCR_MODE=skip_noarchive PAPERLESS_OCR_LANGUAGE=deu PAPERLESS_FILENAME_FORMAT={created_year}/{correspondent}/{title} PAPERLESS_FORCE_SCRIPT_NAME=/paperless
```
and a pod file, tying it all together:
```ini
# paperlessngx.pod
[Pod]
PublishPort=10.88.0.1:8000:8000

[Install]
WantedBy=default.target
```

Like most applications, Paperless-ngx only has a Docker Compose file in its documentation. Instead of manually converting these to plain old Podman Quadlet files, we can leverage [Podlet](https://github.com/containers/podlet), which can convert docker-compose-files to podman .container and .pod files via e.g.
```bash
podlet compose --pod  docker-compose.sqlite.yml
```
The resulting files require minimal adjustments to include, e.g., the right hostnames and auto-updating.

#### Rootless

The other feature I cherish is true rootless containers. For Paperless-ngx, this looks like:
```bash
$ podman top paperlessng-web  uid huser pid  comm
UID         HUSER       PID         COMMAND
0           1444        1           s6-svscan
0           1444        16          s6-supervise
0           1444        17          s6-linux-init-s
0           1444        30          s6-supervise
0           1444        31          s6-supervise
0           1444        32          s6-supervise
0           1444        33          s6-supervise
0           1444        34          s6-supervise
0           1444        35          s6-supervise
0           1444        36          s6-supervise
0           1444        44          s6-ipcserverd
1000        166535      159         [celeryd: celer
1000        166535      161         python3
1000        166535      166         [celery beat] -
1000        166535      174         granian asginl 
1000        166535      194         granian asginl 
1000        166535      250         [celeryd: celer
```
So inside the container namespace, we have several root processes, such as the s6 init and supervisor. The actual application with a Python worker, Granian web server and Celery message queue runs as UID 1000 user without root privileges. On the host system, both users are mapped to different UIDs, the one of the user which executed podman (1444) and one of its sub-UIDs (166535). Neither the application has root privileges inside the container namespace nor would the container's root user have root privileges on the host if it were to escape the container.

#### IPv6

One piece is still missing: IPv6! I don't have a static IP address, so I rely on DynDNS for accessing my server from outside the home network. Since IPv6 doesn't have NAT, I can't just use the host's IPv6 address. Podman doesn't automatically forward IPv6 traffic either, so in [the corresponding Podman issue](https://github.com/containers/podman/issues/4311), the best advice seems to be using socat to just convert all IPv6 traffic to IPv4 internally. This is what I am using:

```ini
# /etc/systemd/system/socat-http.service
[Unit]
Description=Socat http proxy service
Wants=network.target
After=network-online.target

[Service]
ExecStart=/usr/bin/socat TCP6-LISTEN:80,ipv6only=1,fork,su=nobody TCP4:localhost:80

[Install]
WantedBy=multi-user.target default.target
```

and:

```ini
# /etc/systemd/system/socat-https.service
[Unit]
Description=Socat https proxy service
Wants=network.target
After=network-online.target

[Service]
ExecStart=/usr/bin/socat TCP6-LISTEN:443,ipv6only=1,fork,su=nobody TCP4:localhost:443

[Install]
WantedBy=multi-user.target default.target
```

There is probably a more elegant solution to this problem, but I actually have never had any issues in five years or even wasted a thought on this, so why change it.

## Complementary Tools and Considerations

Besides the base OS and the Podman configuration, there are a few additional components I have in place for reducing maintenance and complexity. As a reverse proxy, I prefer [Traefik](https://doc.traefik.io/traefik/) for its excellent documentation, simple Let's Encrypt integration and simple setup (yes, you read that right).

One issue I had to solve was DynDNS. There simply was (or still is?) no DynDNS client available that had proper IPv6 support, so I had to build my own [tiny IPv6 DynDNS updater](https://gitlab.com/lackhove/ddupdate).
A crucial ingredient for a reliable self-hosting setup is of course backups. I run nightly, incremental off-site backups using [duplicacy](https://github.com/gilbertchen/duplicacy) and a [bespoke automation](https://gitlab.com/lackhove/superdup). But both topics are probably worth separate articles.


## Future Work and Final Thoughts

As with any self-hosting setup, you are never finished. One thing I should probably get sorted out is moving `mdadm` notifications into a container. I am already using Scrutiny for monitoring my HDDs' SMART status, so right now I just hope for someone to implement [the feature request](https://github.com/AnalogJ/scrutiny/issues/415). Another issue is Nextcloud, which is one of the most important applications I host but requires attention due to something breaking every few months.

I am not going to bore you with a list of services I am running. Instead, I think it's more interesting to mention what I am not running: Email and this website. Why? Email is just too essential and at the same time too complicated to properly set up and maintain, especially for my whole family. Instead, I just rely on the service by my domain registrar. Similarly, I don't see the benefit in hosting a public website where everything is public anyway, so you have downloaded this HTML from GitHub Pages. This is the final building block to reducing complexity in self-hosting: Knowing what not to host.
