# Documentation - AI Assistant Guide

> **Note:** This file documents the documentation project specifically.
> Always read the [Thermoquad Organization CLAUDE.md](../CLAUDE.md) first
> for organization-wide structure and conventions.

## Overview

Sphinx documentation for the Thermoquad project, styled after Zephyr's documentation.

## Building

Always generate HTML after making changes:

```bash
task html
```

To preview locally:

```bash
task serve
```

**Important:** When adding new pages or changing the toctree structure, clean the
build directory first to ensure navigation updates on all pages:

```bash
task clean && task html
```

Incremental builds (`task html` alone) only regenerate changed pages, so existing
pages may show stale navigation links.

## Structure

```
documentation/
├── source/
│   ├── conf.py              # Sphinx configuration
│   ├── index.rst            # Landing page toctree
│   ├── index.html           # Landing page card layout
│   ├── _static/             # CSS, JS, images
│   ├── _templates/          # Custom Jinja templates
│   ├── introduction/        # Project introduction
│   ├── getting-started/     # Setup guides
│   ├── specifications/      # Protocol and system specs
│   ├── firmware/            # Helios, Slate docs
│   ├── hardware/            # Board documentation
│   ├── tools/               # Heliostat, etc.
│   ├── reference/           # API, glossary, specs
│   └── contributing/        # Contribution guidelines
└── build/                   # Generated output (not committed)
```

## Fusain Specifications

The Fusain protocol specification is the canonical reference for all implementations:

- **Location:** `source/specifications/fusain/`
- **Contents:**
  - `overview.rst` - Protocol overview and device roles
  - `packet-format.rst` - Packet structure, CRC, byte stuffing
  - `messages.rst` - All message types and payloads
  - `packet-payloads.rst` - Detailed payload structures
  - `implementation.rst` - Implementation guide
  - `communication-patterns.rst` - Telemetry, commands, discovery
  - `physical-layer.rst` - LIN, RS-485, plain UART
  - `tcp.rst` / `websocket.rst` - Network transports
  - `packet-routing.rst` - Multi-hop routing

## Related Code

- **Fusain Library (C):** `modules/lib/fusain/` - Embedded C implementation for Zephyr RTOS
  - See `modules/lib/fusain/CLAUDE.md` for API documentation
- **Fusain Library (Go):** `tools/heliostat/pkg/fusain/` - Reference Go implementation
  - Canonical Go implementation used by Heliostat analyzer
  - See `tools/heliostat/CLAUDE.md` for documentation

## CI/CD

Documentation is automatically built and published to GitHub Pages on every push
to the `master` branch.

**Workflow:** `.github/workflows/publish-documentation.yml`

The CI pipeline:
1. Installs Task using `go-task/setup-task@v1`
2. Installs Python dependencies from `requirements.txt`
3. Runs `task html` to build the documentation
4. Deploys `build/html` to GitHub Pages

## Conventions

- Use reStructuredText (.rst) for all documentation
- Follow Zephyr-style formatting (bold headers, italic component names)
- Keep the toctree in index.rst in sync with index.html card links
- Run `task html` before committing to verify build succeeds

---

## AI Assistant Operations

To reload all organization CLAUDE.md files or run a content integrity check, see the **CLAUDE.md Reload** and **Content Integrity Check** sections in the [Thermoquad Organization CLAUDE.md](../CLAUDE.md).

---

**Last Updated:** 2026-01-09
