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

**Trigger:** Scheduled (Mondays at 18:00 UTC) or manual

Performs maintenance on all self-hosted runners by:

- Clearing npm cache
- Uninstalling all Playwright browsers
- Reinstalling dependencies and Playwright

### Check SD Card Health

**Trigger:** Manual (`workflow_dispatch`)

Runs the Raspberry Pi SD card benchmark test on all runners to check for
potential SD card degradation or failures.

### Runner Stats

**Trigger:** Manual (`workflow_dispatch`)

Gathers system information from all runners including:

- System overview (kernel, OS, architecture)
- CPU and memory usage
- Disk usage and inode stats
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

# NVM
# https://github.com/nvm-sh/nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
bash
nvm install 22
```

### Setup GitHub Action

Go to https://github.com/organizations/clusterflick/settings/actions/runners/new?arch=arm64&os=linux

Follow the instructions to install and set up. In the CLI:

- Leave runner group default
- Name: `self-hosted-pi4-X` replacing X with the next index (assuming this is a new Pi 4 runner)
- Additional labels: `pi4,self-hosted-pi4-X`
- Leave work folder default

Then run:
```sh
sudo ./svc.sh install
sudo ./svc.sh status
sudo ./svc.sh start
```

### Setup Tailscale

Run the following and follow instructions for authenticating, approving and managing key.

```sh
# From https://tailscale.com/download/linux
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

### Add to maintenance GitHub Actions

Update the workflows in this repository to include the new runner in any matrix
configurations or runner lists as needed.