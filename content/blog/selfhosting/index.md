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
OpenSUSE was a little late to the immutable container host OS party, but MicroOS combines the best of my then favorite Operating Systems: The minimalsism and immutability of Fedora CoreOS and the roling release model of Archlinux (I use Arch btw.). Without major OS Release upgrades, the system has been absolutely rock solid for me. This is the setup have been running since summer 2020.


## Lessons Learned

The main thing i learned in 13 years of self-hosting is that when moving from hosting a simple file server for yourself to a range of critical services for family and friends, the top priorities should be reliability and simplicity. You really dont want to start debugging your ansible playbooks when and update made the livng room lights go haywire on christmas eve. My current setup is heavily designed to 
* **maximize inherent security:** I dont have the time to closely monitor the system, so the likelyhood of a security issue is minimized by following best practives and keeping all software up to date automatically. In addition, SELINUX and podman rootless containers reduce the impact of a security breach. Needless to say that the number of services reachable from the internet is minimized.
* **minimize the number of components:** Fewer moving parts means fewer parts that might break. I decided against using hypervisors such as proxmox, IaaS systems such as ansible, orchestration such as kubernetes and container management tools such as Portainer. Just a minimal, immutable bare metal OS and a bunch of rootless podman containers. No more than two points of failure for most services.
* **minimize maintenance:** I want to be the one who decided when i spend time on my home server, not some release schedule of the software i am running. So the server should stay running mostly by itself.
* **minimize complexity:** I log on to the server every few months, so i probably wont remember the super complex service-configuration  or the utterly clever solution i implemented. KISS, so avoid unnesseary software with huge config files or components with poor documentation or few users.
* **maxize reliability:** Systems tend to fail at the most inconvenient moments, so indeally, they shouldnt fail at all. Without redundant harware or a UPS, this goal is quite unrealistic, so i try to minimize the impact of failures and enure that disaster recovery is swift and simple.

## The Setup







