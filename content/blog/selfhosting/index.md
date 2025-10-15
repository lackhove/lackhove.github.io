+++
title = "Low Complexity Selfhosting"
date = 2025-10-10
+++

## History

When the Raspberry Pi launched in 2012, i was lucky enought to snatch one of the first batches. This is when i caught the selfhosting bug. From this moment on, my requirements, experience and setup evolved simulateneously. As far as i recall, the milestones were the follwoinng:

#### Raspberry Pi with single 3.5" HDD and Raspbian on the SD-Card
Basically the vanilla setup, hosting ssh and nfs servers. SSH was reachable from the internet via Dyn-DNS. Great for offloading data from my Laptops tiny SSD, but horribly unrealiable since the sd-card kept failing every few weeks.

#### Raspberry Pi with single 3.5" HDD and Archlinux-ARM on a read-only SD-Card
At some point in time, archlinux-ARM gained a feature which allowed mounting the root-system read-only and a handful of tmpfs mounts. The few services it provided were installed from the OS-repo and running as plain systemd-services. This was pretty solid and better yet, very robust to powert outages. 

#### Raspberry Pi with single 3.5" HDD and Archlinux-ARM on a read-only SD-Card with Docker
When the usecases and therefore the list of hosted services grew, i was facing two problems: First, every service is an additional attack-vector, so seperating them as much as possible is probably a good idea. Second, installiung and detting up services, databases etc. is tedious and boring. At that time, docker seemed to adress both issues. Sure, from a security POV, containers cant provide a full-blown isolation, but it still beats running everything with root-permissions on the same host. 

#### x86 mini-ITX server with RAID 1, CentOS and Docker
While at that time, the Raspberry Pi was a great device, i was facing three issues:
* The CPU was slow and the RAM was small
* The supply for pre-built container images for ARM was short, building the images on the Pi was slow
* The USB port was too slow to handle ethernet and two harddrives, a dealbreaker for RAID1
Hence, the next evolution was an ASrock mini-ITX board with an Intel N3150 CPU, two 3.5" HDDs in a RAID1 and CentOS on an SSD. The hardware was more of a disruption than an evolution, the RAID1 made the system much more reliable and with CentOS and docker, i had a rock-solid OS that i had plenty of experience with from work. This one was a keeper, i hardly modified it for a few years, only swapping or adding a few containerized services every now and then.

#### x86 mini-ITX server with RAID 1, CentOS and Podman
While containers are great, docker has its issues. Running the docker daemon as root and the lacking integration with the underlying OS felt like the implementation wasnt using its full potential. The podman came along. Admittedly, the transition wasnt as smooth as advertised (IPv6 gave me quite a few headaches, but moree of that later), but first class integration with systemd and the ability to run containers as non-root users were woth the trouble.

#### x86 mini-ITX server with RAID 1, Fedora Atomic Host / Fedora CoreOS
While the last seup workde great, keeping the host OS up to date, manually fixing issues and wondering how fast i could really restore the system afer a catrastrophic failure seemed more like a having a server as pet than as cattle. Fedora Atomic Host and later Fedora CoreOS solved these issues by having immutable root file systems and performing atomic updats, promising a secure system, that could be safely and autmatically updated at short intervals. The prmise was mostly kept, except for major distro version upgrades.

#### x86 mini-ITX server with RAID 1, OpenSUSE MicroOS
OpenSUSE was a little late to the immutable container host OS party, but MicroOS combines the best of my then favorite Operating Systems: The minimalsism and immutability of Fedora CoreOS and the rolling release model of Archlinux (I use Arch btw). Without major OS Release upgrades, the system has been absolutely rock solid for me. This is the setup have been running since summer 2020.


## Lessons Learned

The main thing i learned in 13 years of self-hosting is that when moving from hosting a simple file server for yourself to a range of critical services for family and friends, the top priorities should be reliability and simplicity. You really dont want to start debugging your k8s setup when and update made the livng room lights go haywire on christmas eve. My current setup is heavily designed to 
* **maximize inherent security:** I dont have the time to closely monitor the system, so the likelyhood of a security issue is minimized by following best practives and keeping all software up to date automatically. In addition, SELINUX and podman rootless containers reduce the impact of a security breach. Needless to say that the number of services reachable from the internet is minimized.
* **minimize the number of components:** Fewer moving parts means fewer parts that might break. I decided against using hypervisors such as proxmox, IaaS systems such as ansible, orchestration such as kubernetes and container management tools such as Portainer. Just a minimal, immutable bare metal OS and a bunch of rootless podman containers. No more than two points of failure for most services.
* **minimize maintenance:** I want to be the one who decided when i spend time on my home server, not some release schedule of the software i am running. So the server should stay running mostly by itself.
* **minimize complexity:** I log on to the server every few months, so i probably wont remember the super complex service-configuration  or the utterly clever solution i implemented. KISS, so avoid unnesseary software with huge config files or components with poor documentation or few users.
* **maxize reliability:** Systems tend to fail at the most inconvenient moments, so indeally, they shouldnt fail at all. Without redundant harware or a UPS, this goal is quite unrealistic, so i try to minimize the impact of failures and enure that disaster recovery is swift and simple.
or in short: **minimize complexity**.


## The Setup

### Level 0: The Hardware

As mentionend earlier, the hardware is a mini-ITX server with an Intel N100 low power CPU, a cheap SSD and two 3.5" HDDs. What i like about the Asus mainborad is that it doesnt have a fan, so one less mechanical part that might fail. The more interestin part is the [PiKVM](https://pikvm.org/) attached to the server, which gives me full control of the system, even on the road: I can push the power and reset buttons, see the screen, acces the and even virtually plug ISO images into the USB drive. In my case, this has several advantages, compared to e.g. virtualization platforms:
* I have full access to the system, even when powered off and before boot
* The server can operate without the PiKVM, so a failure doesnt bring the whole system down
* The PiKVM is only accessible from within my home network (or wireguard when not at home), so one less attack vector i need to worry about
* Its absolutely robust and low maintenance with a read-only file system and hardly any configuration.

### Level 1: The Operating System

#### Basic setup

After installation, the system can be configured as any other (container) Linux OS. However, with its transactional nature, most changes will only be applied after a reboot or a
```bash
$ transactional-update apply
```
which switches into the next snapshot immediately. This command should also be applied after each alteration to the current snapshot, because new snapshot are always derived from the current one, overwritigng changes made to the next one in line. Installing packages (also with `transactional-update`) will automatically create a new snapshot, so going back to the current, working state is as easy as selecting a different snapshot at boot or running `transactional-update rollback`. 

Since i am trying to minimize complexity here,  try to only configure the most basic stuff, such as:
* timezone, keyboard and locale
* ssh setup with `PermitRootLogin without-password`
* Additional File system mounts
* mail and mdadm notifications (I havent found a good contaiern for this task yet)
everything else should go into a container and for sysadmin tasks, OpenSUSE recommends using distrobox (which ive never used though). 

Did i mention the key feature of MicroOS was reliable, autmatic updates? These must be enabled manually via
```bash
$ systemctl enable --now transactional-update.timer
```
and setting the desired reboot interval in `/etc/systemd/system/transactional-update.timer.d/local.conf` and method in `/etc/transactional-update.conf`. I opted for daily and `REBOOT_METHOD=systemd`, since its the simplest one. The system can even determine whether its healthy and will trigger a roll back automaticylly if not. In all those years i never had to do a manual rollback.

Since the base system is so simple, there is no need for complex orchestration tools (ansible, salt, puppet, ...). All relevant configuration is included in the output of 
```bash
$ zypper packages --userinstalled
```
which lists all manually installed packages and the content of the `/etc` dir. Both can be backed up and restored easily.

#### Podman base setup

Most functionality is implemented in in dedicated containers, so we need to set up podman properly. The OS comes with reasonable defaults, but for my usecase i want to keep all images up to date automatically. Podman already ships the required features, which are enabled by:
```bash
$ systemctl enable --now podman-auto-update.timer
```
This wil produce lots of unused images, which we can automatically clean up by adding custom units:
```ini
# /etc/systemd/system/podman-prune.service
[Unit]
Description=Podman prune service
Wants=network.target
After=network-online.target

[Service]
ExecStart=/usr/bin/podman system prune -a --volumes -f

[Install]
WantedBy=multi-user.target default.target`
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
which can be enabled with
```bash
$ systemctl daemon-reload
$ systemctl enable --now podman-prune.timer
```

Since we want to run most services rootless, we also want to set up our (system) user. Since i want to be able to login and look around, i created a normal user with uid 1000
```bash
$ useradd --add-subids-for-system -u 1000 zonk
```
We then mae the podman maintenance tasks available to users with
```bash
$ cp /usr/lib/systemd/system/podman-auto-update.* /etc/systemd/user
$ cp /etc/systemd/system/podman-prune.* /etc/systemd/user
```
After logging in as the new user, we first want to make sure that they can run systemd services in backgroud:
```bash
$ su datenhaufen
$ loginctl enable-linger $USER
```
and enable our two podman tasks
```bash
$ systemctl --user daemon-reload
$ systemctl --user enable --now podman-auto-update.timer
$ systemctl --user enable --now podman-prune.timer
```
Now we are ready to add our first container

### Level 2: Containers


#### Basic setup

With podman, all we need to manage our container lifecycle is systemd, so again, no need for complex contaienr management or orchestration systems such as docker-compose or k8s. Since the integration of quadlet into podman, we dont even need to manually write, manage and update systemd units anymore. Instead, all our containers are managed via simpe ini files in `/etc/container`. The corresponding systemd unit files are created at boot (or `systemctl --user daemon-reload`) and even the users containers are managed right in the `/etc` dir, which further simplifies backup and restore. A simple example would be:
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
Isnt that beatiful? Here i defined that, amongst other things the container image should be auto updated by our `podman-auto-update` service, publish ports and mount volumes with correct SELINUX labels and auto adjusted permissions. 


For multi container applications, we can leverage podmans pods, which basically allow network communication between containers over localhost ports, saving us from the complexity of container networks. [karakeep](https://karakeep.app/) for instance, requires a chome and a meiliseeach container in addition to the application itself. So for this, i have the following container files:
* ```ini
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
* ```ini
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
* ```ini
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
As most applications, karakeep only has a docker-compose file in its documentation. Instead of manually converting these to plain old podman quadlet files, we can leverage [podlet](https://github.com/containers/podlet) which can _"Convert a (docker) compose file to [...] A Quadlet .pod file and .container files."._ The resulting files require minimal adjustments to include e.g. the right hostnames and auto updating.



#### Web Access

An application such as karakeep sshould be accessible from the internet, so we need a reverse proxy. I prefer [traefik](https://doc.traefik.io/traefik/) for its excellent documentation and its simple setup (Yes, you read that right). Mixing reverse proxy and  container configs by adding container labels is not for me, so i just went with an oldschool traefik config: 
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
Again, this is beatifully explicit and simple: We tell traefik where to find our dynamic configs, set ports and timeouts and letsencrypt. For each service, we tell it to redirect http requests to the https router, and pass them to port 8012 of the host machine. Thats it! 

My only pevee here is that in order to access all services and privileged ports, i am currently running this container as root. This can probably be fixed easily but i still havent found the time oir motivation.


#### IPv6

Now one pice is till missing, IPv6! I dont have a static IP address, so i rely on DynDNS for accessing my server from outside the home network. Since IPv6  doesnt have NAT, i cant just use the hosts IPv6 address. Podman doenst automatically forward IPv6 traffic also, so in  [the corresponding podman issue](https://github.com/containers/podman/issues/4311) the best advice seems to be using socat to just convert all IPv6 traffic to IPv4 internally. This is what i am using:
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
and
```ini
#/etc/systemd/system/socat-https.service
[Unit]
Description=Socat http proxy service
Wants=network.target
After=network-online.target

[Service]
ExecStart=/usr/bin/socat TCP6-LISTEN:443,ipv6only=1,fork,su=nobody TCP4:localhost:443

[Install]
WantedBy=multi-user.target default.target
```
There is probably a more elegant solution to this problem, but i actually have never had any issues in five years or even wasted a thought on this, so why change it.

The next issue was related to DynDNS. There simply was (or still is?) no DynDNS client available that had proper IPv6 support, so i had to build my own [tiny IPv6 dyndns updater](https://gitlab.com/lackhove/ddupdate), but this is probably worth a separate article.


## Next Steps

As with any selfhosting setups, you are never finished. 


...