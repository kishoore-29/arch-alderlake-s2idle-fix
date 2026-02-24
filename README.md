# arch-alderlake-s2idle-fix

Fix for suspend/resume crashes and overheating during sleep on HP Victus (Alder Lake) running Arch Linux using s2idle. Stabilizes system by tuning Intel CPU C-states.

## Overview
This document describes a stable solution for suspend/resume crashes and overheating during sleep on HP Victus laptops running Arch Linux with 12th Gen Intel Alder Lake processors using `s2idle` (Modern Standby).

### Tested System Configuration
* **Laptop:** HP Victus
* **CPU:** Intel Core i5-12450H (12th Gen Alder Lake)
* **GPU:** NVIDIA RTX 3050 Mobile
* **Distribution:** Arch Linux
* **Sleep Mode:** `s2idle` (Modern Standby)

To verify that your system only supports `s2idle`, run:
```bash
cat /sys/power/mem_sleep
```
If the output shows only:
```bash
[s2idle]
```
Then S3 deep sleep is not available in your firmware and the system relies entirely on Modern Standby.

## The Problem
Standard kernel configurations often lead to a "Second Sleep Crash" on Alder Lake HP systems.

### Observed Behavior

- First Suspend Cycle: Works normally.
- Second Suspend Cycle: Causes a black screen upon resume; system becomes unresponsive.
- Hardware Watchdog: Triggers an automatic reboot after the resume failure.
- Overheating: The laptop remains hot during sleep, leading to rapid battery drain.
- Logs: Typically indicate watchdog activity or ACPI timeout after resume failure.

### Root Cause

On certain HP firmware implementations, deep CPU C-states (C6, C8, and C10) conflict with the s2idle transition. This causes resume handshake failures and ACPI restore instability.

This issue is not caused by:

- NVIDIA GPU drivers
- NVMe power management
- Dual-boot configurations
- Desktop environment (GNOME/KDE/Sway)

Alder Lake aggressively enters deep package C-states under s2idle, which exposes firmware resume bugs on some HP implementations.

## Why C2 Is the Correct Balance
Because s2idle does not fully power down the system like S3, it depends on CPU idle states to reduce power consumption.

- C1: Too shallow. Resume is stable, but the system remains warm and drains battery quickly.
  
- C6+: Too deep. Causes resume instability and watchdog-triggered reboots.
  
- C2: Balanced. Prevents entry into problematic deep idle states while maintaining stable resume and acceptable thermals.

# Implementation Guide

### 1. Edit GRUB Configuration
Open your GRUB configuration file:
```bash
sudo nano /etc/default/grub
```
Locate the line starting with GRUB_CMDLINE_LINUX_DEFAULT and add the parameter intel_idle.max_cstate=2 inside the quotes.
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_idle.max_cstate=2"
```
### 2. Rebuild GRUB
Apply the changes by regenerating the GRUB configuration:
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
### 3. Reboot
Restart your system to apply the kernel parameters:
```bash
  reboot now
```
# Verification
After rebooting, perform the following checks to ensure the fix is active:

Check Kernel Command Line:
```bash
cat /proc/cmdline
```
Confirm intel_idle.max_cstate=2 is present in the output.

## Stress Test Suspend:

- Suspend and resume the system 3–4 times consecutively.
- Leave the system suspended for 20 minutes to confirm it remains cool.

### Important Notes
S3 deep sleep is not available on this firmware; do not use mem_sleep_default=deep.

NVIDIA and NVMe were not the root causes of this specific crash.

This solution is specific to Alder Lake systems exhibiting resume instability.

