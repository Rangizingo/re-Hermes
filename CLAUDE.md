# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Reverse-engineering workspace for the Hermes Agent (Nous Research, MIT). The repo holds writeups and captured evidence only. There is no application code to build. Read `README.md` first for scope and known gaps.

## Commands

None yet. The repo has no build, lint or test tooling. Don't add any unless asked.

## Targets (read-only)

- Source: `/Users/peterb/Documents/AI/hermes-agent`, pinned at commit `97a9ae8ee5` (2026-10-11). Upstream clone, origin `github.com/nousresearch/hermes-agent`. This is the analysis target.
- Installed reference: `/Users/peterb/.hermes/hermes-agent`, build `d604141` (2026-07-21). `hermes --version` reports v0.19.0. It is a git repo. Use it to compare against the source, and cite it for anything claimed about what runs on this Mac.

The installed build is older than the source pin, so behavior can differ between them. Say which tree a claim comes from.

## Layout

- `docs/`: writeups, one topic per file. Architecture notes go here. None exist yet, so don't assume any.
- `evidence/`: captured output that the writeups cite.

## Rules

- Keep every write inside this repo. Never edit either Hermes tree.
- Cite `file:line` in the target tree for every behavior claim, and the commit it was read at.
- Label estimates as estimated. Label anything not run as not run.
- REA's binary tools need Ghidra. Ghidra 12.1.4 is installed through Homebrew. For REA runs, set `GHIDRA_INSTALL_DIR=/opt/homebrew/opt/ghidra/libexec` (already in `~/.zshrc`). Hermes is Python, so readable source is the main artifact.
- The compliance hook flags 19 upstream files (47 queue entries) for SQL string concatenation. They are upstream code. Review them, don't edit them.
