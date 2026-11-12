# Self-Hosted Workflows

> **[Clusterflick](https://clusterflick.com)** is a free, open-source site
> listing every film showing in London, from 450+ venues in one search.
>
> This repository holds the GitHub Actions workflows that run on Clusterflick's
> own self-hosted runners. See the
> [Clusterflick organisation](https://github.com/clusterflick) for how the
> pieces fit together.

GitHub Actions workflows designed to run on self-hosted runners for the
Clusterflick project.

## Overview

This repository contains workflows that require self-hosted runners, primarily
for tasks that benefit from running on dedicated infrastructure or need to be
run from a residential IP address.

## Self-Hosted Runners

The runners are two Optiplex boxes, `delrunners` and `delrunners2`, with eight
runners each (`self-hosted-optiplex-N` and `self-hosted-optiplex5090-N`). Every
runner goes out through the same 5G router.

x86 mini PCs can also host several runners each; see
[Setting Up an x86 Runner Host](#setting-up-an-x86-runner-host).

## Workflows

### Diagnose Runner Network

**Trigger:** Manual (`workflow_dispatch`)

Checks the shared 5G link from whichever runner picks the job up, and writes a
table to the job summary:

- DNS, the route out, latency and packet loss to 1.1.1.1, and path MTU
- `git ls-remote` against a download of `clusterflick/scripts` from
  codeload.github.com, which is how npm fetches it: listing refs is a few KB, so
  a link that passes that but crawls on the ~1 MB download is slow rather than
  down
- Single-connection download speed from Cloudflare (`download_mb` input, 10 MB
  by default)

A run uses about 11 MB of the data allowance.

It uses no actions, so it still starts when the link is too slow for a normal
job's setup. If the link is slow, the
[Restart Router](https://github.com/clusterflick/data-retrieved/actions/workflows/restart-router.yml)
workflow in data-retrieved reconnects it.

## Usage

### Running Workflows Manually

1. Navigate to the **Actions** tab in the repository
2. Select the workflow you want to run
3. Click **Run workflow**
4. Select the branch and click **Run workflow**

## Setting Up a New Runner

### Get up to date

```sh
sudo apt update
sudo apt upgrade
```

### Install known dependencies

```sh
# Git
sudo apt install git -y

# Fio
sudo apt install fio -y

# Smartctl
sudo apt install smartmontools -y

# NVM
# https://github.com/nvm-sh/nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
bash
# Match `.node-version` in clusterflick/scripts. Jobs get their own Node from
# actions/setup-node, so this is what manual runs on the box use - but it must
# still be at least 22.12, below which `require()` of an ESM package (such as
# camoufox-js) throws.
nvm install 24
```

### Install browser dependencies

Camoufox ships its own patched Firefox but relies on the system libraries
Playwright's Firefox needs, and on Xvfb for its virtual display.

```sh
npx playwright@latest install-deps firefox
sudo apt install xvfb -y
```

### Set up the SSD

```sh
# Partition & format the SSD
sudo wipefs -a /dev/sda
sudo parted /dev/sda --script mklabel gpt mkpart primary ext4 0% 100%
sudo mkfs.ext4 -F /dev/sda1

# fstab entries
sudo blkid /dev/sda1 # get the UUID from this put
```

Update `/etc/fstab` with:

```
UUID=<uuid>  /mnt/runner-work  ext4  defaults,noatime,nofail,x-systemd.device-timeout=10  0  2
/mnt/runner-work  /home/alib/actions-runner/_work  none  bind,nofail,x-systemd.requires=/mnt/runner-work  0  0
```

And then run

```sh
# Fix Ownership
sudo chown -R alib:alib /mnt/runner-work
```

### Set up TRIM

```sh
sudo tee /usr/local/sbin/enable-ssd-trim.sh > /dev/null <<'EOF'
#!/bin/sh
echo unmap > /sys/block/sda/device/scsi_disk/0:0:0:0/provisioning_mode
EOF
sudo chmod 755 /usr/local/sbin/enable-ssd-trim.sh
```

```sh
sudo tee /etc/systemd/system/ssd-trim-enable.service > /dev/null <<'EOF'
[Unit]
Description=Enable TRIM (unmap) on USB SSD
After=local-fs.target
Wants=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/enable-ssd-trim.sh

[Install]
WantedBy=multi-user.target
EOF
```

Enable, start, and test the new service

```sh
sudo systemctl daemon-reload
sudo systemctl enable ssd-trim-enable.service
sudo systemctl start ssd-trim-enable.service
```

And then run

```sh
sudo udevadm control --reload-rules
sudo udevadm trigger
lsblk --discard # shows sda DISC-MAX non-zero
sudo fstrim -v /mnt/runner-work
```

Confirm everything is set up:

```sh
findmnt /mnt/runner-work                 # on /dev/sda1
findmnt ~/actions-runner/_work           # on /dev/sda1
lsblk --discard                          # sda DISC-MAX non-zero
sudo fstrim -v /mnt/runner-work          # reports bytes trimmed
sudo systemctl status actions.runner.*   # active, "Listening for Jobs"
```

### Runner environment

Update `~/actions-runner/.env` with:

```
RUNNER_CACHE_ROOT=/mnt/runner-work
```

### Install the Camoufox browser

Some retrieves through Camoufox, whose browser is a ~650MB download that is
**not** installed by `npm install`. The data pipeline never fetches it —
`clusterflick/.github/setup-cache-paths` points `CAMOUFOX_INSTALL_DIR` at the
SSD, and the retrieve job fails fast if the browser isn't there.

Install it by hand, into the same directory the jobs will use, from any repo
that depends on `clusterflick/scripts`:

```sh
export CAMOUFOX_INSTALL_DIR="$RUNNER_CACHE_ROOT/camoufox"
./node_modules/.bin/camoufox-js fetch
node -e 'require("camoufox-js/dist/pkgman.js").launchPath()'
```

The last line is the check that matters: `camoufox-js fetch` takes the newest
GitHub release including prereleases, and a broken build has shipped before,
extracting fonts and config with no browser binary. `launchPath()` resolves the
binary itself and throws when it isn't there.

The install directory is always a plain function of the runner's own
`RUNNER_CACHE_ROOT`, so a host with several runners gets one install each.
Repeat the above per runner.

### Confirm EEPROM boot order

```sh
rpi-eeprom-config
```

The boot order must not put USB first (`0xf14`). Use the SD-first default (no
explicit `BOOT_ORDER` line, or an explicit `0xf41`).

### Passwordless sudo

```sh
sudo visudo -f /etc/sudoers.d/runner-monitoring
```

and paste in

```
alib ALL=(ALL) NOPASSWD: /usr/sbin/fstrim, /usr/sbin/smartctl, /usr/local/sbin/enable-ssd-trim.sh

```

### Setup GitHub Action

Go to
https://github.com/organizations/clusterflick/settings/actions/runners/new?arch=arm64&os=linux

Follow the instructions to install and set up. In the CLI:

- Leave runner group default
- Name: `self-hosted-pi4-X` replacing X with the next index (assuming this is a
  new Pi 4 runner)
- Additional labels: `pi4,self-hosted-pi4-X`
- Leave work folder default

Then run:

```sh
sudo ./svc.sh install
sudo ./svc.sh status
sudo ./svc.sh start
```

### Setup Tailscale

Run the following and follow instructions for authenticating, approving and
managing key.

```sh
# From https://tailscale.com/download/linux
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

## Setting Up an x86 Runner Host

A small x86 machine (e.g. a Dell OptiPlex Micro) running Ubuntu Server 24.04
LTS, hosting several runners at once. Most of the Pi setup above applies; the
differences are the system NVMe in place of a USB SSD, and several runners
sharing one host.

Each browser-based retrieve needs roughly 3-4GB of RAM, so size the number of
runners to the machine's memory and cores (e.g. 4 runners on 6 cores / 16GB),
then adjust from what real runs show.

### Get up to date and install known dependencies

As for a Pi: see [Get up to date](#get-up-to-date) and
[Install known dependencies](#install-known-dependencies).

Ubuntu Server also needs a build toolchain, which Raspberry Pi OS and macOS
already have: some dependencies (such as `better-sqlite3`, used by
`data-analysed`) compile native code during `npm install`, and fail with
`gyp ERR! stack Error: not found: make` without it.

```sh
sudo apt install build-essential -y
```

### Install browser system libraries

These are the system packages the browsers need (installed with apt), not the
browsers themselves — those are downloaded into the cache root in
[Pre-install the browsers](#pre-install-the-browsers) below. Install the
libraries for both browsers the retrieves use: Chromium (through Playwright) and
Firefox (for Camoufox), plus Xvfb.

```sh
npx playwright@latest install-deps chromium firefox
sudo apt install xvfb -y
```

### Create the cache root

The NVMe is the system disk, so there is no partitioning, fstab entry or bind
mount — each runner's `_work` stays in its own directory. Create one cache root,
shared by every runner on the host:

```sh
sudo mkdir -p /srv/runner-cache
sudo chown alib:alib /srv/runner-cache
```

Sharing it means one Chromium and one Camoufox install per host rather than one
per runner. Once installed, jobs only read them, and the npm cache is safe to
use concurrently. Install both browsers by hand (below) **before** starting the
runners, so several jobs never race to install into the same path.

### TRIM

Not needed: NVMe TRIM works natively, and Ubuntu's `fstrim.timer` runs weekly by
default (`systemctl status fstrim.timer`).

The **Reset dependencies on runners** workflow still calls
`/usr/local/sbin/enable-ssd-trim.sh`, so create a no-op at that path before
adding this host to its matrix:

```sh
sudo tee /usr/local/sbin/enable-ssd-trim.sh > /dev/null <<'SCRIPT'
#!/bin/sh
# No-op on x86 hosts: the NVMe supports TRIM natively. Exists because the
# "Reset dependencies on runners" workflow calls this path on every runner.
exit 0
SCRIPT
sudo chmod 755 /usr/local/sbin/enable-ssd-trim.sh
```

### Passwordless sudo

Jobs on self-hosted runners install Chromium without `--with-deps` (eight
runners running apt-get at once fought over the dpkg lock), so the libraries
from [Install browser system libraries](#install-browser-system-libraries) have
to be on the host before the first job. Passwordless sudo is still worth
granting for maintenance and for anything that runs `sudo` unattended. Grant it
outright — the same as the Pis get from Raspberry Pi OS's default
`/etc/sudoers.d/010_pi-nopasswd`:

```sh
sudo visudo -f /etc/sudoers.d/runner-monitoring
```

and paste in

```
alib ALL=(ALL) NOPASSWD: ALL

```

Check it the way Playwright will use it:

```sh
sudo -k; sudo -n sh -c true && echo ok
```

### Pre-install the browsers

The installs run from a repo that depends on `clusterflick/scripts`, so clone
one. The repos are public, so HTTPS works without setting up a key:

```sh
git clone https://github.com/clusterflick/data-retrieved.git ~/data-retrieved
cd ~/data-retrieved
npm install
```

Then, from that directory, download both browsers into the cache root:

```sh
export RUNNER_CACHE_ROOT=/srv/runner-cache
export PLAYWRIGHT_BROWSERS_PATH="$RUNNER_CACHE_ROOT/playwright-browsers"
export CAMOUFOX_INSTALL_DIR="$RUNNER_CACHE_ROOT/camoufox"

npx playwright install chromium
./node_modules/.bin/camoufox-js fetch
node -e 'require("camoufox-js/dist/pkgman.js").launchPath()'
```

As in [Install the Camoufox browser](#install-the-camoufox-browser), the last
line is the check that matters: it throws if the fetched release has no browser
binary.

### Setup GitHub Actions runners

Go to
https://github.com/organizations/clusterflick/settings/actions/runners/new?arch=x64&os=linux

For each runner `N` on the host, unpack the runner into its own directory,
following the download commands on that page but in `~/actions-runner-N` rather
than `~/actions-runner`. Each runner needs its own registration token, so reload
the page between runners. In the CLI:

- Leave runner group default
- Name: `self-hosted-<host>-N` (e.g. `self-hosted-optiplex-1`)
- Additional labels: `<host>,self-hosted-<host>-N`
- Leave work folder default

Update `~/actions-runner-N/.env` with:

```
RUNNER_CACHE_ROOT=/srv/runner-cache
```

Then, in each `~/actions-runner-N`:

```sh
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```

### BIOS

Set the machine to power back on after a power cut (on Dell: **Power Management
→ AC Recovery → Power On**).

### Setup Tailscale

As for a Pi: see [Setup Tailscale](#setup-tailscale). Use the host name from the
runner names (e.g. `sudo tailscale up --hostname=optiplex`), and disable key
expiry for the machine in the Tailscale admin console.
