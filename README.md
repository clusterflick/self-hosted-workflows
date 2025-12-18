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

### Check Venue Statuses

**Trigger:** Manual (`workflow_dispatch`)

Validates cinema venue IDs across multiple cinema chains:

| Venue        | Runner                 |
| ------------ | ---------------------- |
| Cineworld    | Self-hosted            |
| Curzon       | Ubuntu (GitHub-hosted) |
| Everyman     | Ubuntu (GitHub-hosted) |
| Vue          | Self-hosted            |
| Odeon        | Self-hosted            |
| Omniplex     | Ubuntu (GitHub-hosted) |
| Picturehouse | Self-hosted            |

### Free Space on Runners

**Trigger:** Scheduled (Mondays at 12:00 UTC) or manual

Performs maintenance on all self-hosted runners by cleaning the npm cache to
reclaim disk space.

## Usage

### Running Workflows Manually

1. Navigate to the **Actions** tab in the repository
2. Select the workflow you want to run
3. Click **Run workflow**
4. Select the branch and click **Run workflow**
