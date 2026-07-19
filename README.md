# Linux Resource Investigation Runbook

This repository records a command-by-command Linux OS-level investigation flow for memory, CPU, process lifecycle, and resource spike analysis.

The goal is to build an expert-level operational runbook using built-in Linux tools first, without installing external utilities.

## Scope

- Process lifecycle and resource usage
- Node-level memory consumption
- Node-level CPU consumption
- Per-process CPU and memory usage
- Timeline creation for spikes and dips
- Evidence-based interpretation of why resource usage changed

## Current Focus

Memory investigation using built-in Linux commands.

## Memory Investigation

### Step 1: Node Memory Summary

Command:

```bash
free -wh
