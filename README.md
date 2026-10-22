# Self-Hosted Workflows

GitHub Actions workflows designed to run on self-hosted runners for the
Clusterflick project.

## Overview

This repository contains workflows that require self-hosted runners, primarily
for tasks that benefit from running on dedicated infrastructure or need to be
run from a residential IP address.

## Self-Hosted Runners

The workflows in this repository target a cluster of Raspberry Pi 4 devices
configured as GitHub Actions self-hosted runners. Each runner is labeled with
its specific identifier (e.g., `self-hosted-pi4-1`).

## Workflows

### Reset Dependencies on Runners

**Trigger:** Scheduled or manual

Performs maintenance on all self-hosted runners by:

- Clearing npm cache
- Uninstalling all Playwright browsers
- Reinstalling dependencies and Playwright
- Installing the Camoufox browser if it is missing, and failing if the install
  it finds is incomplete

This is how the Pi runners get their Camoufox browser: it is a separate ~650MB
download that `npm install` does not fetch. It is only fetched when absent, so
running this workflow never upgrades a working install. The matrix covers the
Pis only — the macOS runners are provisioned by hand.

### Check runner storage health

**Trigger:** Manual (`workflow_dispatch`)

Runs a series of checks to confirm the harddrive health, as well as confirm the
runner setup is correctly using the SSD for all work and cache requirements.

### Check SD Card Health (Deprecated)

**Trigger:** Manual (`workflow_dispatch`)

Runs the Raspberry Pi SD card benchmark test on all runners to check for
potential SD card degradation or failures.

**Deprecated:** Storage has been moved to SSDs. This workflow is still
available, but is now deprecated.

### Runner Stats

**Trigger:** Manual (`workflow_dispatch`)

Gathers system information from all runners including:

- System overview (kernel, OS, architecture)
- CPU and memory usage
- SSD usage and stats
- SD card usage and stats
- SD card health checks
- Temperature readings
- Process counts

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

On a Pi, run the **Reset dependencies on runners** workflow; it installs
Camoufox as part of its run. To do it by hand instead — or on the macOS runners,
which that workflow doesn't cover — use the same install directory the jobs will
use, from any repo that depends on `clusterflick/scripts`:

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

### Add to maintenance GitHub Actions

Update the workflows in this repository to include the new runner in any matrix
configurations or runner lists as needed.
