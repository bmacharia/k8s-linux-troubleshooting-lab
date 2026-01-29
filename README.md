# Kubernetes & Linux Troubleshooting Lab

This repository documents a 12-week hands-on troubleshooting curriculum
focused on Linux internals, container runtimes, and Kubernetes failure modes.

The goal is not deployment, but diagnosis.

## What This Repo Demonstrates

- Linux-first debugging mindset
- Hypothesis-driven troubleshooting
- Kubernetes failure domain isolation
- Real-world incident response patterns

## Lab Environment

- Raspberry Pi cluster
- K3s
- containerd
- Prometheus + Grafana
- CoreDNS
- Local-path / Longhorn storage

## Structure

Each failure scenario includes:

- Controlled break
- Observable symptoms
- Investigation path
- Root cause
- Minimal fix
- Postmortem

This mirrors real production incident workflows.

## Why This Matters

Most outages are not solved by redeploying.
They are solved by understanding systems under stress.

This repo is a record of learning to do exactly that.
