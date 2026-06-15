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
nvm install 22
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

### Set up TRIM udev rule

Update `/etc/udev/rules.d/10-trim-usb.rules` with:

```
ACTION=="add|change", ATTRS{idVendor}=="174c", ATTRS{idProduct}=="55aa", SUBSYSTEM=="scsi_disk", ATTR{provisioning_mode}="unmap"
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
alib ALL=(ALL) NOPASSWD: /usr/sbin/fstrim, /usr/sbin/smartctl
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
