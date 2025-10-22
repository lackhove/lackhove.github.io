+++
title = "From Hobby to Chore and Back in 13 years: My Low-Complexity Self-Hosting Stack with OpenSUSE MicroOS and Podman Rootless Containers"
date = 2025-10-10
+++



## History

When the Raspberry Pi launched in 2012, I was lucky enough to snatch one of the first batches. This is when I caught the selfhosting bug. From this moment on, my requirements, experience, and setup evolved significantly, but not always in snyc. As it turns out, the requirements as a student with plenty of spare time and hardly any responsibilities were quite different to having a family and a day job thirteen years later.
As far as I recall, the milestones were the following:

**Raspberry Pi with single 3.5" HDD and Raspbian on the SD-Card**
Basically the vanilla setup, hosting SSH and NFS servers. SSH was reachable from the internet via DynDNS.  The few services it provided were installed from the OS repository and running as plain systemd services. Great for offloading data from my laptop's tiny SSD, but horribly unreliable since the SD-card kept failing every few weeks.

**Raspberry Pi with single 3.5" HDD and Arch Linux-ARM on a read-only SD-Card**
I countered the failing SD-Cards by switching to Arch Linux ARM which at that time was able to mount the root file system read-only, significantly reducing SD Card wear and making the system immune to power outages. A first step towards minimizing maintenance.

**Raspberry Pi with single 3.5" HDD and Arch Linux-ARM on a read-only SD-Card with Docker**
When the use cases and therefore the list of hosted services grew, I was facing two problems: 
* First, every service is an additional attack vector, so i wanted to improve separation. 
* Second, installing and setting up services, databases, etc. is tedious and boring. 

At that time, Docker seemed a way out of configuration complexity while also improving isolation

**x86 mini-ITX server with RAID 1, CentOS and Docker**
While the Raspberry Pi was great at that time, I was facing two issues:
* The supply of pre-built container images for ARM was short, and building images on the Pi was slow
* The single hard drive was a significant reliablility risk

Hence, the next evolution was an ASRock mini-ITX board with an Intel N3150 CPU, two 3.5" HDDs in a RAID 1 and CentOS on an SSD. The hardware was more of a disruption than an evolution as it allowed for RAID 1, making the system much more reliable. With CentOS and Docker, I had a rock-solid OS that I had plenty of experience with from work. This one was a keeper; I hardly modified it for a few years, only swapping or adding a few containerized services now and then.

**x86 mini-ITX server with RAID 1, CentOS and Podman**
While containers are great, Docker has its issues. Running the Docker daemon as root is sub-par from a security POV and and the lacking integration with the underlying OS made management cumbersome. Then Podman came along, solving both issues with a traditional fork/exec model, tight systemd integration and the ability to run rootless containers, i.e. containers without privileged permisions.

**x86 mini-ITX server with RAID 1, Fedora Atomic Host / Fedora CoreOS**
While the last setup worked great, disadvantages were:
* OS updates required manual intervention
* The root file system was readable, so potential power outages could wreck my server. 

The setup still felt more like a pet than cattle. Fedora Atomic Host and later Fedora CoreOS solved these issues by performing atomic updates, i.e. updateing the OS as a whole and enabling simpe roll-backs in case of problems. It also brought back the immutable root file system, offering better (not perfect) protection from power outages without the cost and complexity of a UPS. Overall, the setup lived up to the promise of a a secure, always  up-to-date and reliable system that could survive most power outages.


**x86 mini-ITX server with RAID 1, openSUSE MicroOS**
My only peevee with Fedora CoreOS was the slow updates due to ostree and that it broke and required manual intervention at both major OS verson upgrades i installed. Therefore, I was delighted to read the announcement of OpenSUSE MicroOS, which is basically an immutable container host system like Fedora CoreOS but with BTRFS instead of ostree and - tadaa -  a rolling release model! Without major OS release upgrades, the system has been absolutely rock solid for me. This is the setup I have been running since summer 2020.


## Lessons Learned

The main thing I learned in 13 years of self-hosting is that when moving from hosting a simple file server for yourself to a range of critical services for family and friends, the top priorities should be reliability and simplicity. You really don't want to start debugging your Kubernetes setup when an update made the living room lights go haywire on Christmas Eve. My current setup is heavily designed to:

* **Maximize inherent security:** I don't have the time to closely monitor the system, so the likelihood of a security issue is minimized by following best practices and keeping all software up to date automatically. In addition, SELinux and Podman rootless containers reduce the impact of a security breach. Needless to say, the number of services reachable from the internet is minimized.

* **Minimize the number of components:** Fewer moving parts means fewer parts that might break. I decided against using hypervisors such as Proxmox, IaaS systems such as Ansible, orchestration such as Kubernetes, and container management tools such as Portainer. Just a minimal, immutable bare metal OS and a bunch of rootless Podman containers. No more than two points of failure for most services.

* **Minimize maintenance:** I want to be the one who decides when I spend time on my home server, not some release schedule of the software I am running. So the server should stay running mostly by itself.

* **Minimize complexity:** I log on to the server every few months, so I probably won't remember the super complex service configuration or the utterly clever solution I implemented. KISS, so avoid unnecessary software with huge config files or components with poor documentation or few users.

* **Maximize reliability:** Systems tend to fail at the most inconvenient moments, so ideally, they shouldn't fail at all. Without redundant hardware or a UPS, this goal is quite unrealistic, so I try to minimize the impact of failures and ensure that disaster recovery is swift and simple.

Or in short: **minimize complexity**.


## The Setup

### Level 0: The Hardware

As mentioned earlier, the hardware is a mini-ITX server with an Intel N100 low-power CPU, a cheap SSD, and two 3.5" HDDs. What I like about the ASUS motherboard is that it doesn't have a fan, so one less mechanical part that might fail. The more interesting part is the [PiKVM](https://pikvm.org/) attached to the server, which gives me full control of the system, even on the road: I can push the power and reset buttons, see the screen, access the BIOS, and even virtually plug ISO images into the USB drive. In my case, this has several advantages compared to, e.g., virtualization platforms:

* I have full access to the system, even when powered off and before boot
* The server can operate without the PiKVM, so a failure doesn't bring the whole system down
* The PiKVM is only accessible from within my home network (or Wireguard when not at home), so one less attack vector I need to worry about
* It's absolutely robust and low-maintenance with a read-only file system and hardly any configuration.

### Level 1: The Operating System

#### Basic setup

After installation, the system can be configured like any other container Linux OS. However, with its transactional nature, most changes will only be applied after a reboot or:

```bash
$ transactional-update apply
```

This command switches into the next snapshot immediately. You should also apply this command after each alteration to the current snapshot, because new snapshots are always derived from the current one, overwriting changes made to the next one in line. Installing packages (also with `transactional-update`) will automatically create a new snapshot, so going back to the current, working state is as easy as selecting a different snapshot at boot or running `transactional-update rollback`.

Since I am trying to minimize complexity here, I try to only configure the most basic stuff, such as:

* timezone, keyboard, and locale
* SSH setup with `PermitRootLogin without-password`
* additional file system mounts
* mail and mdadm notifications (I haven't found a good container for this task yet)

Everything else should go into a container. For sysadmin tasks, openSUSE recommends using distrobox (which I've never used, though).

Did I mention the key feature of MicroOS was reliable, automatic updates? These must be enabled manually via:

```bash
$ systemctl enable --now transactional-update.timer
```

Set the desired reboot interval in `/etc/systemd/system/transactional-update.timer.d/local.conf` and method in `/etc/transactional-update.conf`. I opted for daily and `REBOOT_METHOD=systemd`, since it's the simplest. The system can even determine whether it's healthy and will trigger a rollback automatically if not. In all those years, I've never had to do a manual rollback.

Since the base system is so simple, there is no need for complex orchestration tools (Ansible, Salt, Puppet, etc.). All relevant configuration is included in the output of:

```bash
$ zypper packages --userinstalled
```

This lists all manually installed packages and the content of the `/etc` directory. Both can be backed up and restored easily.

#### Podman base setup

Most functionality is implemented in dedicated containers, so we need to set up Podman properly. The OS comes with reasonable defaults, but for my use case, I want to keep all images up to date automatically. Podman already ships the required features, which are enabled by:

```bash
$ systemctl enable --now podman-auto-update.timer
```

This will produce lots of unused images, which we can automatically clean up by adding custom units:

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

These can be enabled with:

```bash
$ systemctl daemon-reload
$ systemctl enable --now podman-prune.timer
```

Since we want to run most services rootless, we also need to set up our (system) user. Since I want to be able to login and look around, I created a normal user with UID 1000:

```bash
$ useradd --add-subids-for-system -u 1000 zonk
```

We then make the Podman maintenance tasks available to users with:

```bash
$ cp /usr/lib/systemd/system/podman-auto-update.* /etc/systemd/user
$ cp /etc/systemd/system/podman-prune.* /etc/systemd/user
```

After logging in as the new user, we first want to make sure they can run systemd services in the background:

```bash
$ su zonk
$ loginctl enable-linger $USER
```

and enable our two Podman tasks:

```bash
$ systemctl --user daemon-reload
$ systemctl --user enable --now podman-auto-update.timer
$ systemctl --user enable --now podman-prune.timer
```

Now we are ready to add our first container.

### Level 2: Containers

#### Basic setup

With Podman, all we need to manage our container lifecycle is systemd, so again, no need for complex container management or orchestration systems such as Docker Compose or Kubernetes. Since the integration of Quadlet into Podman, we don't even need to manually write, manage, and update systemd units anymore. Instead, all our containers are managed via simple INI files in `/etc/containers/systemd`. The corresponding systemd unit files are created at boot (or `systemctl --user daemon-reload`), and even the user's containers are managed right in the `/etc` directory, which further simplifies backup and restore. A simple example would be:

```ini
# /etc/containers/systemd/users/1000/mosquitto.container
[Service]
Restart=on-failure
TimeoutStopSec=70

[Container]
ContainerName=mosquitto
Image=docker.io/eclipse-mosquitto:2
AutoUpdate=image
PublishPort=1883:1883
PublishPort=9001:9001
Volume=/var/lib/docker-confs/mosquitto/config:/mosquitto/config/:Z,U
Volume=/var/lib/docker-confs/mosquitto/data:/mosquitto/data/:Z,U
Volume=/var/lib/docker-confs/mosquitto/log:/mosquitto/log/:Z,U

[Install]
WantedBy=default.target
```

Isn't that beautiful? Here I defined that, among other things, the container image should be auto-updated by our `podman-auto-update` service, publish ports, and mount volumes with correct SELinux labels and auto-adjusted permissions.

For multi-container applications, we can leverage Podman's pods, which basically allow network communication between containers over localhost ports, saving us from the complexity of container networks. [Karakeep](https://karakeep.app/), for instance, requires a Chrome and a Meilisearch container in addition to the application itself. So for this, I have the following container files:

```ini
# /etc/containers/systemd/users/1000/karakeep-chrome.container
[Service]
Restart=on-failure
TimeoutStopSec=70

[Container]
Pod=karakeep.pod
ContainerName=karakeep-chrome
AutoUpdate=image
Image=gcr.io/zenika-hub/alpine-chrome:123
Exec=--no-sandbox --disable-gpu --disable-dev-shm-usage '--remote-debugging-address=0.0.0.0' '--remote-debugging-port=9222' --hide-scrollbars
Environment=KARAKEEP_VERSION=release
```

```ini
# /etc/containers/systemd/users/1000/karakeep-meilisearch.container
[Service]
Restart=on-failure
TimeoutStopSec=70

[Container]
Environment=MEILI_NO_ANALYTICS=true
Image=docker.io/getmeili/meilisearch:v1.11.1
AutoUpdate=image
Pod=karakeep.pod
ContainerName=karakeep-meilisearch
Volume=/var/lib/docker-confs/karakeep/meilisearch:/meili_data:Z
```

```ini
# /etc/containers/systemd/users/1000/karakeep-web.container
[Service]
Restart=on-failure
TimeoutStopSec=70

[Container]
Environment=MEILI_ADDR=http://localhost:7700 BROWSER_WEB_URL=http://localhost:9222 DATA_DIR=/data KARAKEEP_VERSION=release NEXTAUTH_SECRET=<redacted> DISABLE_SIGNUPS=true MEILI_MASTER_KEY=<redacted> NEXTAUTH_URL=<redacted> OPENAI_API_KEY=<redacted>
Image=ghcr.io/karakeep-app/karakeep:release
Pod=karakeep.pod
ContainerName=karakeep-web
AutoUpdate=image
Volume=/var/lib/docker-confs/karakeep/data:/data:Z
```

and a pod file, tying it all together at `/etc/containers/systemd/users/1000/karakeep.pod` with:

```ini
[Pod]
PublishPort=8012:3000

[Install]
WantedBy=default.target
```

Like most applications, Karakeep only has a Docker Compose file in its documentation. Instead of manually converting these to plain old Podman Quadlet files, we can leverage [Podlet](https://github.com/containers/podlet), which can "convert a (Docker) Compose file to a Quadlet .pod file and .container files." The resulting files require minimal adjustments to include, e.g., the right hostnames and auto-updating.

#### Web Access

An application such as Karakeep should be accessible from the internet, so we need a reverse proxy. I prefer [Traefik](https://doc.traefik.io/traefik/) for its excellent documentation and simple setup (yes, you read that right). Mixing reverse proxy and container configs by adding container labels is not for me, so I just went with an old-school Traefik config:

```yaml
# /var/lib/docker-confs/traefik/traefik.yml
providers:
  file:
    directory: /etc/traefik/dynamic/

entryPoints:
  web:
    address: ":80"
    transport:
      respondingTimeouts:
        readTimeout: 600
  websecure:
    address: ":443"
    transport:
      respondingTimeouts:
        readTimeout: 600

certificatesResolvers:
  letsencrypt:
    acme:
      email: <redacted>
      storage: /etc/traefik/acme.json
      httpChallenge:
        entryPoint: web
```

and a bunch of dynamic configs:

```toml
# /var/lib/docker-confs/traefik/dynamic/karakeep.toml
[http]
  [http.middlewares]
    [http.middlewares.karakeep-redirect.redirectScheme]
        scheme = "https"

  [http.routers.karakeep-redirect]
    entrypoints = ["web"]
    rule = "Host(`<redacted>`)"
    middlewares = ["karakeep-redirect"]
    service = "karakeep"

  [http.routers.karakeep]
    entrypoints = ["websecure"]
    rule = "Host(`<redacted>`)"
    service = "karakeep"
    [http.routers.karakeep.tls]
        certResolver = "letsencrypt"

  [http.services]
    [http.services.karakeep.loadBalancer]
      [[http.services.karakeep.loadBalancer.servers]]
        url = "http://host.containers.internal:8012/"
```

Again, this is beautifully explicit and simple: We tell Traefik where to find our dynamic configs, set ports and timeouts, and Letsencrypt. For each service, we tell it to redirect HTTP requests to the HTTPS router and pass them to port 8012 of the host machine. That's it!

My only peeve here is that in order to access all services and privileged ports, I am currently running this container as root. This can probably be fixed easily, but I still haven't found the time or motivation.

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

The next issue was related to DynDNS. There simply was (or still is?) no DynDNS client available that had proper IPv6 support, so I had to build my own [tiny IPv6 DynDNS updater](https://gitlab.com/lackhove/ddupdate), but this is probably worth a separate article.


## Next Steps

As with any self-hosting setup, you are never finished.

...
