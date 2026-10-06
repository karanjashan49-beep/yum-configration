# Configuring YUM on Linux (RHEL-family)

A practical guide to setting up package repositories in two ways:

1. **Manually**: writing `.repo` files yourself (local ISO, HTTP/FTP, or remote mirror)
2. **With Subscription Manager**: registering a RHEL system with Red Hat and enabling entitled repos

> **Note:** On RHEL 8/9 and derivatives, `yum` is a symlink to `dnf`. All commands below work with either, and repo files use the same format.

---

## Table of Contents

- [How YUM finds repositories](#how-yum-finds-repositories)
- [Part 1: Manual configuration](#part-1-manual-configuration)
  - [Anatomy of a .repo file](#anatomy-of-a-repo-file)
  - [Option A: Local repo from an ISO / DVD](#option-a-local-repo-from-an-iso--dvd)
  - [Option B: Remote repo over HTTP/HTTPS/FTP](#option-b-remote-repo-over-httphttpsftp)
  - [Option C: Build and serve your own repo](#option-c-build-and-serve-your-own-repo)
  - [Adding a repo with dnf config-manager](#adding-a-repo-with-dnf-config-manager)
- [Part 2: Subscription Manager](#part-2-subscription-manager)
  - [Register the system](#register-the-system)
  - [Attach a subscription](#attach-a-subscription)
  - [List and enable repositories](#list-and-enable-repositories)
  - [Using an activation key](#using-an-activation-key)
  - [Unregister / clean up](#unregister--clean-up)
- [Verifying and managing repos](#verifying-and-managing-repos)
- [Troubleshooting](#troubleshooting)
- [Quick reference](#quick-reference)

---

## How YUM finds repositories

YUM/DNF reads repository definitions from:

| Location | Purpose |
|---|---|
| `/etc/yum.repos.d/*.repo` | One or more repo definitions per file |
| `/etc/yum.conf` or `/etc/dnf/dnf.conf` | Global options (gpgcheck, proxy, etc.) |
| `/etc/yum.repos.d/redhat.repo` | **Auto-generated** by Subscription Manager. Do not edit by hand |

---

## Part 1: Manual configuration

### Anatomy of a .repo file

```ini
[repo-id]
name=Human readable description
baseurl=file:///path/or/http://server/path
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```

| Field | Meaning |
|---|---|
| `[repo-id]` | Unique ID, no spaces |
| `name` | Description shown in `repolist` |
| `baseurl` | Location of the repo (`file://`, `http://`, `https://`, `ftp://`) |
| `mirrorlist` / `metalink` | Alternative to `baseurl`: a URL returning a list of mirrors |
| `enabled` | `1` = on, `0` = off |
| `gpgcheck` | `1` = verify package signatures (recommended) |
| `gpgkey` | Path/URL of the public key used to verify packages |

---

### Option A: Local repo from an ISO / DVD

Useful for offline systems or labs. RHEL 8/9 ISOs contain **two** repositories: `BaseOS` and `AppStream`.

**1. Mount the media**

```bash
# From a DVD/virtual CD drive
sudo mkdir -p /mnt/cdrom
sudo mount /dev/sr0 /mnt/cdrom

# OR from an ISO file
sudo mount -o loop,ro /path/to/rhel-9.x-x86_64-dvd.iso /mnt/cdrom
```

**2. Make the mount persistent** (add to `/etc/fstab`)

```
/path/to/rhel-9.x-x86_64-dvd.iso  /mnt/cdrom  iso9660  loop,ro  0 0
```

Test with `sudo mount -a`.

**3. Create the repo file**

```bash
sudo vi /etc/yum.repos.d/local.repo
```

```ini
[local-baseos]
name=Local BaseOS
baseurl=file:///mnt/cdrom/BaseOS
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[local-appstream]
name=Local AppStream
baseurl=file:///mnt/cdrom/AppStream
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```

**4. Verify**

```bash
sudo dnf clean all
sudo dnf repolist
```

> On RHEL 7 and older there is a single repo at the ISO root, so use `baseurl=file:///mnt/cdrom`.

---

### Option B: Remote repo over HTTP/HTTPS/FTP

```ini
# /etc/yum.repos.d/custom.repo
[custom-repo]
name=Custom Remote Repo
baseurl=http://repo.example.com/rhel9/BaseOS/
enabled=1
gpgcheck=1
gpgkey=http://repo.example.com/RPM-GPG-KEY-custom
```

For a repo you fully trust and that has no signing key (e.g. an internal lab), you can set `gpgcheck=0`. **Avoid this on production systems.**

---

### Option C: Build and serve your own repo

Create a repo from a folder of RPMs and share it over HTTP.

```bash
# 1. Install tools
sudo dnf install -y createrepo_c httpd

# 2. Put RPMs in a directory
sudo mkdir -p /var/www/html/myrepo
sudo cp *.rpm /var/www/html/myrepo/

# 3. Generate metadata
sudo createrepo_c /var/www/html/myrepo

# 4. Start the web server
sudo systemctl enable --now httpd
sudo firewall-cmd --permanent --add-service=http && sudo firewall-cmd --reload
```

On clients:

```ini
[myrepo]
name=My Internal Repo
baseurl=http://<server-ip>/myrepo
enabled=1
gpgcheck=0
```

After adding or removing RPMs, re-run `createrepo_c --update /var/www/html/myrepo`.

---

### Adding a repo with dnf config-manager

```bash
sudo dnf install -y dnf-plugins-core

# Add a repo from a URL to a .repo file
sudo dnf config-manager --add-repo https://example.com/path/to/example.repo

# Enable / disable an existing repo
sudo dnf config-manager --set-enabled repo-id
sudo dnf config-manager --set-disabled repo-id
```

---

## Part 2: Subscription Manager

On a **RHEL** system, repos are delivered through your Red Hat subscription. `subscription-manager` registers the machine, and generates `/etc/yum.repos.d/redhat.repo` for you.

### Register the system

```bash
sudo subscription-manager register --username <your-rh-username>
# You'll be prompted for the password
```

Check status:

```bash
sudo subscription-manager status
sudo subscription-manager identity
```

### Attach a subscription

Many accounts now use **Simple Content Access (SCA)**, where registering is enough and no manual attach is required. If your account does **not** use SCA:

```bash
# Let the system pick a suitable subscription
sudo subscription-manager attach --auto

# Or list what's available and attach a specific pool
sudo subscription-manager list --available
sudo subscription-manager attach --pool=<POOL_ID>

# Show what is attached
sudo subscription-manager list --consumed
```

### List and enable repositories

```bash
# All repos your subscription gives access to
sudo subscription-manager repos --list

# Only the enabled ones
sudo subscription-manager repos --list-enabled

# Enable repos (RHEL 9 example, x86_64)
sudo subscription-manager repos \
  --enable=rhel-9-for-x86_64-baseos-rpms \
  --enable=rhel-9-for-x86_64-appstream-rpms

# Disable a repo
sudo subscription-manager repos --disable=<repo-id>
```

Typical repo IDs:

| RHEL | BaseOS | AppStream |
|---|---|---|
| 8 | `rhel-8-for-x86_64-baseos-rpms` | `rhel-8-for-x86_64-appstream-rpms` |
| 9 | `rhel-9-for-x86_64-baseos-rpms` | `rhel-9-for-x86_64-appstream-rpms` |

Then confirm:

```bash
sudo dnf clean all
sudo dnf repolist
```

### Using an activation key

Handy for automation (Kickstart, Ansible, cloud-init):

```bash
sudo subscription-manager register --org=<ORG_ID> --activationkey=<KEY_NAME>
```

### Unregister / clean up

```bash
sudo subscription-manager remove --all     # remove subscriptions
sudo subscription-manager unregister       # unregister from Red Hat
sudo subscription-manager clean            # wipe local subscription data
```

---

## Verifying and managing repos

```bash
dnf repolist              # enabled repos
dnf repolist all          # enabled and disabled repos
dnf repolist -v           # detailed info
dnf makecache             # rebuild metadata cache
dnf clean all             # clear cached data

dnf search <package>
dnf info <package>
dnf install <package>

# Use or skip a specific repo for one command
dnf --enablerepo=<repo-id> install <package>
dnf --disablerepo=<repo-id> update
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `This system is not registered with an entitlement server` | RHEL system not registered | `subscription-manager register` |
| `Repository ... is listed more than once` | Duplicate repo ID in `/etc/yum.repos.d/` | Make each `[repo-id]` unique |
| `Cannot find a valid baseurl for repo` | Wrong URL, no network, or ISO not mounted | Check `baseurl`, `ping`/`curl` the URL, run `mount` |
| `repomd.xml ... No such file` | `baseurl` doesn't point to the directory containing `repodata/` | Point to the folder that contains `repodata` |
| `GPG check FAILED` / `Public key ... not installed` | Missing or wrong `gpgkey` | `rpm --import <key-file>` or fix `gpgkey=` |
| `Curl error (60) SSL certificate problem` | Untrusted or expired cert, or a proxy | Fix CA trust, check the system date, check the proxy |
| Old metadata or stale results | Cache out of date | `dnf clean all && dnf makecache` |
| Repo not showing in `repolist` | `enabled=0`, or typo in file extension | Make sure the file ends in `.repo` and `enabled=1` |

Useful diagnostics:

```bash
sudo subscription-manager refresh
sudo journalctl -u rhsmcertd
sudo tail -f /var/log/rhsm/rhsm.log
dnf repolist -v
```

---

## Quick reference

```bash
# --- Manual ---
sudo vi /etc/yum.repos.d/myrepo.repo
sudo dnf clean all && sudo dnf repolist

# --- Subscription Manager ---
sudo subscription-manager register
sudo subscription-manager attach --auto        # skip if SCA is enabled
sudo subscription-manager repos --list
sudo subscription-manager repos --enable=<repo-id>
sudo dnf repolist
```

---

## Notes

- Commands were written for RHEL 8/9. On CentOS Stream, Rocky Linux or AlmaLinux, use Part 1 (they don't use Subscription Manager).
- Always keep `gpgcheck=1` for repos you don't fully control.
- Don't edit `/etc/yum.repos.d/redhat.repo`. Use `subscription-manager repos` instead.
