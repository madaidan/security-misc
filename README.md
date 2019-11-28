# enhances misc security settings #

Inspired by Kernel Self Protection Project (KSPP)

* Implements most if not all recommended Linux kernel settings (sysctl) and
kernel parameters by KSPP.

* https://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project

kernel hardening:

* deactivates Netfilter's connection tracking helper
Netfilter's connection tracking helper module increases kernel attack
surface by enabling superfluous functionality such as IRC parsing in
the kernel. (!) Hence, this package disables this feature by shipping the
/etc/modprobe.d/30_nf_conntrack_helper_disable.conf configuration file.

* Kernel symbols in various files in /proc are hidden as they can be
very useful for kernel exploits.

* Kexec is disabled as it can be used to load a malicious kernel.
/etc/sysctl.d/kexec.conf

* ASLR effectiveness for mmap is increased.

* The TCP/IP stack is hardened by disabling ICMP redirect acceptance,
ICMP redirect sending and source routing to prevent man-in-the-middle attacks,
ignoring all ICMP requests, enabling TCP syncookies to prevent SYN flood
attacks and enabling RFC1337 to protect against time-wait assassination
attacks.

* Some data spoofing attacks are made harder.

* SACK can be disabled as it is commonly exploited and is rarely used by
uncommenting settings in file /etc/sysctl.d/tcp_sack.conf.

* Slab merging is disabled as sometimes a slab can be used in a vulnerable
way which an attacker can exploit.

* Sanity checks, redzoning, and memory poisoning are enabled.

* Machine checks (MCE) are disabled which makes the kernel panic
on uncorrectable errors in ECC memory that could be exploited.

* Kernel Page Table Isolation is enabled to mitigate Meltdown and increase
KASLR effectiveness.

* SMT is disabled as it can be used to exploit the MDS and other
vulnerabilities.

* All mitigations for the MDS vulnerability are enabled.

* A systemd service clears System.map on boot as these contain kernel symbols
that could be useful to an attacker.
/etc/kernel/postinst.d/30_remove-system-map
/lib/systemd/system/remove-system-map.service
/usr/lib/security-misc/remove-system.map

* Coredumps are disabled as they may contain important information such as
encryption keys or passwords.
/etc/security/limits.d/disable-coredumps.conf
/etc/sysctl.d/coredumps.conf
/lib/systemd/coredump.conf.d/disable-coredumps.conf

* The thunderbolt and firewire kernel modules are blacklisted as they can be
used for DMA (Direct Memory Access) attacks.

* IOMMU is enabled with a boot parameter to prevent DMA attacks.

* The kernel now panics on oopses to prevent it from continuing running a
flawed process.

* Bluetooth is blacklisted to reduce attack surface. Bluetooth also has
a history of security concerns.
https://en.wikipedia.org/wiki/Bluetooth#History_of_security_concerns

* A systemd service restricts /proc/cpuinfo, /proc/bus, /proc/scsi and
/sys to the root user only. This hides a lot of hardware identifiers from
unprivileged users and increases security as /sys exposes a lot of information
that shouldn't be accessible to unprivileged users. As this will break many
things, it is disabled by default and can optionally be enabled by running
`systemctl enable hide-hardware-info.service` as root.

Improve Entropy Collection

* Load jitterentropy_rng kernel module.
/usr/lib/modules-load.d/30_security-misc.conf

Uncommon network protocols are blacklisted:
These are rarely used and may have unknown vulnerabilities.
/etc/modprobe.d/uncommon-network-protocols.conf
The network protocols that are blacklisted are:

* DCCP - Datagram Congestion Control Protocol
* SCTP - Stream Control Transmission Protocol
* RDS - Reliable Datagram Sockets
* TIPC - Transparent Inter-process Communication
* HDLC - High-Level Data Link Control
* AX25 - Amateur X.25
* NetRom
* X25
* ROSE
* DECnet
* Econet
* af_802154 - IEEE 802.15.4
* IPX - Internetwork Packet Exchange
* AppleTalk
* PSNAP - Subnetwork Access Protocol
* p8023 - Novell raw IEEE 802.3
* p8022 - IEEE 802.2

user restrictions:

* A systemd service mounts /proc with hidepid=2 at boot to prevent users from
seeing each other's processes.

* The kernel logs are restricted to root only.

* The BPF JIT compiler is restricted to the root user and is hardened.

* The ptrace system call is restricted to the root user only.

restricts access to the root account:

* `su` is restricted to only users within the group `sudo` which prevents
users from using `su` to gain root access or to switch user accounts.
/usr/share/pam-configs/wheel-security-misc
(Which results in a change in file `/etc/pam.d/common-auth`.)

* Add user `root` to group `sudo`. This is required to make above work so
login as a user in a virtual console is still possible.
debian/security-misc.postinst

* Abort login for users with locked passwords.
/usr/lib/security-misc/pam-abort-on-locked-password

* Logging into the root account from a virtual, serial, whatnot console is
prevented by shipping an existing and empty /etc/securetty.
(Deletion of /etc/securetty has a different effect.)
/etc/securetty.security-misc

Protect Linux user accounts against brute force attacks.
Lock user accounts after 50 failed login attempts using pam_tally2.
/usr/share/pam-configs/tally2-security-misc

informational output during Linux PAM:

* Show failed and remaining password attempts.
* Document unlock procedure if Linux user account got locked.
* Point out, that there is no password feedback for `su`.
* Explain locked (root) account if locked.
* /usr/share/pam-configs/tally2-security-misc
* /usr/lib/security-misc/pam_tally2-info
* /usr/lib/security-misc/pam-abort-on-locked-password

access rights restrictions:

* Strong Linux User Account Separation.
Removes read, write and execute access for others for all users who have
home folders under folder /home by running for example
"chmod o-rwx /home/user"
during package installation, upgrade or pam. This will be done only once per
folder in folder /home so users who wish to relax file permissions are free to
do so. This is to protect previously created files in user home folder which
were previously created with lax file permissions prior installation of this
package.
debian/security-misc.postinst
/usr/share/pam-configs/permission-lockdown-security-misc
/usr/lib/security-misc/permission-lockdown

access rights relaxations:

Redirect calls for pkexec to lxqt-sudo because pkexec is incompatible with
hidepid.
https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=860040
https://forums.whonix.org/t/cannot-use-pkexec/8129
/usr/bin/pkexec.security-misc

This package does (not yet) automatically lock the root account password.
It is not clear that would be sane in such a package.
It is recommended to lock and expire the root account.
In new Whonix builds, root account will be locked by package
anon-base-files.
https://www.whonix.org/wiki/Root
https://www.whonix.org/wiki/Dev/Permissions
https://forums.whonix.org/t/restrict-root-access/7658
However, a locked root password will break rescue and emergency shell.
Therefore this package enables passwordless resuce and emergency shell.
This is the same solution that Debian will likely addapt for Debian
installer.
https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=802211
Adverse security effects can be prevented by setting up BIOS password
protection, grub password protection and/or full disk encryption.
/etc/systemd/system/emergency.service.d/override.conf
/etc/systemd/system/rescue.service.d/override.conf

Disables TCP Time Stamps:

TCP time stamps (RFC 1323) allow for tracking clock
information with millisecond resolution. This may or may not allow an
attacker to learn information about the system clock at such
a resolution, depending on various issues such as network lag.
This information is available to anyone who monitors the network
somewhere between the attacked system and the destination server.
It may allow an attacker to find out how long a given
system has been running, and to distinguish several
systems running behind NAT and using the same IP address. It might
also allow one to look for clocks that match an expected value to find the
public IP used by a user.

Hence, this package disables this feature by shipping the
/etc/sysctl.d/tcp_timestamps.conf configuration file.

Note that TCP time stamps normally have some usefulness. They are
needed for:

* the TCP protection against wrapped sequence numbers; however, to
trigger a wrap, one needs to send roughly 2^32 packets in one
minute:  as said in RFC 1700, "The current recommended default
time to live (TTL) for the Internet Protocol (IP) [45,105] is 64".
So, this probably won't be a practical problem in the context
of Anonymity Distributions.
* "Round-Trip Time Measurement", which is only useful when the user
manages to saturate their connection. When using Anonymity Distributions,
probably the limiting factor for transmission speed is rarely the capacity
of the user connection.

Application specific hardening:

* Enables APT seccomp-BPF sandboxing. /etc/apt/apt.conf.d/40sandbox
* Deactivates previews in Dolphin.
* Deactivates previews in Nautilus.
* Deactivates thumbnails in Thunar.
* Enables punycode (`network.IDN_show_punycode`) by default in Thunderbird
to make phising attacks more difficult. Fixing URL not showing real Domain
Name (Homograph attack).

Discussion:

Happening primarily in Whonix forums.
https://forums.whonix.org/t/kernel-hardening/7296
## How to install `security-misc` using apt-get ##

1\. Download [Whonix's Signing Key]().

```
wget https://www.whonix.org/patrick.asc
```

Users can [check Whonix Signing Key](https://www.whonix.org/wiki/Whonix_Signing_Key) for better security.

2\. Add Whonix's signing key.

```
sudo apt-key --keyring /etc/apt/trusted.gpg.d/whonix.gpg add ~/patrick.asc
```

3\. Add Whonix's APT repository.

```
echo "deb https://deb.whonix.org buster main contrib non-free" | sudo tee /etc/apt/sources.list.d/whonix.list
```

4\. Update your package lists.

```
sudo apt-get update
```

5\. Install `security-misc`.

```
sudo apt-get install security-misc
```

## How to Build deb Package ##

Replace `apparmor-profile-torbrowser` with the actual name of this package with `security-misc` and see [instructions](https://www.whonix.org/wiki/Dev/Build_Documentation/apparmor-profile-torbrowser).

## Contact ##

* [Free Forum Support](https://forums.whonix.org)
* [Professional Support](https://www.whonix.org/wiki/Professional_Support)

## Donate ##

`security-misc` requires [donations](https://www.whonix.org/wiki/Donate) to stay alive!
