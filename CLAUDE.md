# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repo ("super-hellthalos") is the GitOps repository and workspace for the user's home Kubernetes cluster. The primary goal is **learning** Kubernetes and related technologies. A working cluster is the means, not the end.

## Working agreements (override global defaults)

- **Caveman mode is OFF in this repo.** Respond in normal, full prose. This overrides the global CLAUDE.md default.
- **Do not create or modify code or config files (manifests, Helm values, Kustomize overlays, scripts, etc.) without explicit permission.** Propose the change, explain it, and wait for a go-ahead. Show snippets inline in chat when explaining.
- **Teach, don't just do.** Explain the *why* behind each concept and design choice, point out trade-offs and alternatives, and relate things back to how Kubernetes works under the hood. Prefer letting the user run commands themselves when it aids understanding (they can use `! <command>`).
- When recommending tools or approaches, account for the hardware constraints below and for the fact that this is a learning setup.

## Hardware

Currently a single compute node:
- Intel i5 CPU
- 16 GB RAM
- 256 GB SSD

Single-node implications to keep in mind: no real HA, control plane and workloads share the node, resource budgets are tight (be deliberate about memory-hungry components), and storage is local-only unless something else is added later.

## Current state

The repository is empty: no cluster distribution, GitOps tool, or directory layout has been chosen yet, and it is not yet a git repository. Update this file as decisions are made (distribution, GitOps controller, repo layout, secrets handling, common commands).
