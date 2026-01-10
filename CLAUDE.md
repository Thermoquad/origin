# Thermoquad Organization - AI Assistant Guide

## Overview

This is the **Thermoquad organization project root**. It contains all tooling, SDKs, libraries, and firmware projects for Thermoquad embedded systems development.

**Important:** This CLAUDE.md provides organization-level conventions and structure. Individual projects (firmwares, libraries) have their own CLAUDE.md files that should be read in conjunction with this one.

---

## Organization Structure

```
Thermoquad/                    # Organization root (this directory)
├── .claude/                   # AI assistant working files (plans, settings)
├── CLAUDE.md                  # Symlink to origin/CLAUDE.md (this file)
├── origin/                    # West manifest repository (version controlled)
│   ├── CLAUDE.md              # This file (canonical location)
│   ├── west.yml               # West manifest for Zephyr workspace
│   ├── Taskfile.dist.yml      # Organization-level task definitions
│   ├── tasks/                 # Shared task scripts
│   ├── documentation/         # Sphinx documentation (public-facing)
│   │   ├── source/            # reStructuredText source files
│   │   └── build/             # Generated HTML (not committed)
│   └── docs/                  # Internal documentation
│       └── protocols/         # Protocol specifications
│           └── serial_protocol.md  # Fusain Protocol v2.0
├── apps/                      # Firmware applications
│   ├── helios/                # Helios burner ICU firmware
│   ├── slate/                 # Slate controller firmware
│   └── [other-firmwares]/     # Future firmware projects
├── modules/                   # Zephyr modules and libraries
│   ├── lib/                   # Shared libraries
│   │   └── fusain/            # Fusain serial protocol library
│   ├── hal/                   # Hardware abstraction layers
│   └── [other-modules]/       # Other module types
├── boards/                    # Custom board definitions
├── zephyr/                    # Zephyr RTOS (managed by west)
└── tools/                     # Development tools and scripts
```

---

## Directory Purposes

### `origin/` - West Manifest Repository
- **Purpose:** Version-controlled repository containing west manifest and org-level configuration
- **Contains:**
  - `west.yml` - Defines workspace dependencies and module locations
  - `Taskfile.dist.yml` - Organization-level build and development tasks
  - `CLAUDE.md` - This file
  - `documentation/` - Sphinx documentation (see below)
  - `docs/protocols/` - Internal protocol specifications (Fusain Protocol v2.0)
  - `.envrc`, `.tool-versions` - Development environment configuration
- **Git:** This is a separate git repository
- **Symlinks:** Configuration files are symlinked to org root for convenience

### `origin/documentation/` - Sphinx Documentation
- **Purpose:** Public-facing documentation built with Sphinx
- **Build:** Run `task html` to generate HTML output
- **Preview:** Run `task serve` to preview locally
- **Style:** Based on Zephyr documentation theme with dark mode support
- **Contents:**
  - Introduction and project overview
  - Specifications (Fusain protocol, burner cycle)
  - Firmware documentation (Helios, Slate)
  - Hardware reference
  - API documentation
- **Documentation:** `origin/documentation/CLAUDE.md`

### `apps/` - Firmware Applications
- **Purpose:** Executable firmware projects for specific hardware platforms
- **Platform:** Zephyr RTOS on Raspberry Pi Pico 2 (RP2350/RP2354)
- **Examples:**
  - `helios/` - Liquid fuel burner ignition control unit (ICU)
  - `slate/` - Controller firmware
- **Convention:** Each app has its own CLAUDE.md that references this file

### `modules/lib/` - Shared Libraries
- **Purpose:** Reusable code modules shared across multiple firmware projects
- **Libraries:**
  - `fusain/` - Fusain serial protocol library (✅ implemented)
- **Usage:** Referenced by firmware applications via CMake or west manifest
- **Documentation:** Each library has its own CLAUDE.md

### `modules/hal/`, `modules/fs/`, etc. - Other Modules
- **Purpose:** Hardware abstraction layers, filesystems, and other Zephyr modules
- **Source:** May be external dependencies or custom implementations

### `boards/` - Custom Board Definitions
- **Purpose:** Board support packages for custom hardware
- **Convention:** Follow Zephyr board definition structure

### `zephyr/` - Zephyr RTOS
- **Purpose:** Zephyr RTOS source code
- **Management:** Managed by west (do not modify directly)

### `tools/` - Development Tools
- **Purpose:** Scripts and utilities for development workflow
- **Examples:** Build scripts, flash utilities, code generators
- **Notable Tools:**
  - `heliostat/` - Fusain protocol analyzer (Go)
    - Real-time packet decoder and error detector
    - Cobra CLI with `raw_log` and `error_detection` commands
    - TUI mode with live statistics (default)
    - **Reference Go implementation:** `pkg/fusain` (canonical Go implementation of Fusain protocol)
    - Usage: `heliostat error_detection --port /dev/ttyUSB0`
    - See `tools/heliostat/CLAUDE.md` for documentation

---

## CLAUDE.md Convention

**Every repository and significant project directory should have its own CLAUDE.md file.**

### Organization-Level CLAUDE.md (This File)
- **Location:** `origin/CLAUDE.md` (symlinked to org root)
- **Purpose:** Document organization structure, conventions, and cross-project patterns
- **Scope:** Applies to all projects in the organization

### Project-Level CLAUDE.md Files
- **Location:** At each project root (e.g., `apps/helios/CLAUDE.md`)
- **Purpose:** Document project-specific architecture, code, and workflows
- **Requirement:** MUST include a header section referencing this shared file

**Example Project CLAUDE.md Header:**
```markdown
# [Project Name] - AI Assistant Guide

> **Note:** This file documents the [Project Name] project specifically.
> Always read the [Thermoquad Organization CLAUDE.md](../../CLAUDE.md) first
> for organization-wide structure and conventions.

## Project Overview
[Project-specific content...]
```

---

## AI Assistant Working Files

### Plan Storage

**Location:** `.claude/` at the organization root (`/home/kazw/Projects/Thermoquad/.claude/`)

When AI assistants create implementation plans for Thermoquad projects, plans should be stored in the organization's `.claude/` directory rather than the user's home directory.

**Contents:**
- `plans/` - Implementation plan files (`.md`)
- `settings.local.json` - Local Claude Code settings

**Why Organization-Level:**
- Plans are project-specific and belong with the project
- Easier to reference across sessions
- Keeps user's home directory clean
- Plans can be shared or reviewed if needed

---

## Development Philosophy

### Ask Questions, Don't Assume

**CRITICAL:** When working on Thermoquad projects, prioritize asking clarifying questions over making assumptions.

- If the requirement is ambiguous, ask for clarification
- If multiple approaches are valid, present options and ask for preference
- If you're unsure about project structure, ask before creating files
- If documentation is unclear or contradictory, ask rather than guessing

**Example:**
- ❌ Bad: "I'll create a new serial library in `src/common/serial/`"
- ✅ Good: "Should I create the serial library in `modules/lib/serial/` (shared) or `apps/helios/src/serial/` (app-specific)?"

---

## Platform & Tooling

### Target Hardware
- **Platform:** Raspberry Pi Pico 2 (Development)
  - RP2350A (Cortex-M33) - Current development boards
  - RP2354A (Cortex-M33F) - Planned production boards
- **RTOS:** Zephyr RTOS

### Build System
- **West:** Zephyr workspace manager
- **CMake:** Build configuration
- **Taskfile:** Task runner (preferred over direct west/cmake commands)

### Development Environment
- **Direnv:** Environment management (`.envrc`)
- **ASDF:** Tool version management (`.tool-versions`)

---

## Build Conventions

### Always Use Taskfile

**IMPORTANT:** Use Taskfile commands for building, not direct `west build` commands.

```bash
task build-firmware    # Build firmware
task flash-firmware    # Flash to device
task rebuild-firmware  # Clean and rebuild in one command
```

**Why:**
- Ensures consistent build environment across projects
- Includes formatting and validation steps
- Used by CI/CD pipelines
- Handles proper command sequencing

### Firmware Flashing Safety

**CRITICAL SAFETY REQUIREMENT - AI ASSISTANTS READ CAREFULLY:**

**NEVER automatically execute `task flash-firmware` or any firmware flashing command.**

Thermoquad systems control safety-critical hardware (burners, pumps, motors). Flashing incorrect firmware can cause:
- Equipment damage
- Fire hazard
- Personal injury

**Required Procedure:**
1. Build firmware with `task build-firmware`
2. Show build output to confirm success
3. **ASK the user to manually flash** the firmware
4. Wait for user to execute `task flash-firmware` themselves
5. Only proceed after user confirms successful flash

**Why This Matters:**
- User must verify build artifacts before deployment
- Physical safety interlock (user presence required)
- Prevents accidental firmware updates during debugging
- Allows user to abort if environment is unsafe

**Remember:** You can build firmware, but you cannot flash it. The user has physical control over when firmware is deployed to hardware.

---

## Code Style & Formatting

### Naming Conventions
- `snake_case` - Functions and variables
- `UPPER_CASE` - Constants and macros
- Descriptive names over abbreviations

### Formatting
- **Formatter:** clang-format (`.clang-format` at project root)
- **Consistency:** Run formatter before committing

### Comments
- Group code into logical blocks with comment headers
- Document "why" not "what"
- Explain safety-critical logic and non-obvious behavior

### File Organization

**Standard Order for C Source Files:**

C source files should be organized in the following order with clearly marked comment sections:

1. **Includes** - All `#include` statements
2. **Config** - Configuration constants and logging registration
3. **State Struct Definition** - Data structures for module state
4. **Static Variables** - File-scope static variables
5. **Forward Declarations** - Function prototypes for internal functions
6. **Public API** - Public functions exposed via headers
7. **Thread Functions** - Thread entry points (if applicable)
8. **Init Functions** - Initialization functions (if any)
9. **Hardware Functions** - Hardware-specific code (e.g., UART polling)
10. **Helper Functions** - Internal helper functions (may have sub-groups)

**Comment Section Format:**

```c
//////////////////////////////////////////////////////////////
// Section Name
//////////////////////////////////////////////////////////////
```

**Example Structure:**

```c
// Includes
#include <zephyr/kernel.h>
#include <project/header.h>

LOG_MODULE_REGISTER(module_name);

//////////////////////////////////////////////////////////////
// Config
//////////////////////////////////////////////////////////////

#define LOOP_SLEEP_MS 100
#define TIMEOUT_MS 30000

//////////////////////////////////////////////////////////////
// State Struct Definition
//////////////////////////////////////////////////////////////

struct module_state {
  bool enabled;
  uint64_t last_update_time;
};

//////////////////////////////////////////////////////////////
// Static Variables
//////////////////////////////////////////////////////////////

static struct module_state state;
static const struct device* uart_dev;

//////////////////////////////////////////////////////////////
// Forward Declarations
//////////////////////////////////////////////////////////////

static void process_data(void);
static int init_hardware(void);

//////////////////////////////////////////////////////////////
// Public API
//////////////////////////////////////////////////////////////

void public_function(void) {
  // Implementation
}

//////////////////////////////////////////////////////////////
// Thread Functions
//////////////////////////////////////////////////////////////

int worker_thread(void) {
  // Thread implementation
}

//////////////////////////////////////////////////////////////
// Init Functions
//////////////////////////////////////////////////////////////

static int init_hardware(void) {
  // Initialization
}

//////////////////////////////////////////////////////////////
// Hardware Functions
//////////////////////////////////////////////////////////////

static void poll_uart(void) {
  // UART polling
}

//////////////////////////////////////////////////////////////
// Helper Functions
//////////////////////////////////////////////////////////////

static void process_data(void) {
  // Data processing
}
```

**Notes:**
- Not all sections are required - use only what's needed for the file
- Sub-sections can be added under "Helper Functions" for organization
- Keep related code together within each section
- Use consistent comment divider format (60 slashes)

---

## Version Control

### Git Workflow

**IMPORTANT: Always get approval before committing changes.**

1. Show changes with `git diff` or `git diff --staged`
2. Explain modifications and rationale
3. Show proposed commit message
4. Wait for explicit approval
5. Only then commit

**Never commit without showing changes first.**

### Commit Style

**Format:** Conventional Commits

```
<type>(<scope>): <subject>

<body>

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

**Types:**
- `feat` - New feature
- `fix` - Bug fix
- `docs` - Documentation only
- `style` - Code style/formatting
- `refactor` - Code refactoring
- `test` - Adding tests
- `chore` - Maintenance tasks

**Scopes:** Use project-specific scopes (e.g., `helios`, `slate`, `serial-lib`)

---

## Project Status

### Active Projects

#### Helios - Burner ICU Firmware
- **Status:** ✅ Active development
- **Location:** `apps/helios/`
- **Platform:** Raspberry Pi Pico 2 (RP2350A)
- **Purpose:** Liquid fuel burner ignition control unit
- **Key Features:** State machine, temperature PID, motor control, serial protocol
- **Documentation:** `apps/helios/CLAUDE.md`

#### Slate - Controller Firmware
- **Status:** ✅ Active development
- **Location:** `apps/slate/`
- **Platform:** Raspberry Pi Pico 2 (RP2350A)
- **Purpose:** Controller with LVGL display for Helios telemetry monitoring
- **Key Features:** LVGL display, serial communication, telemetry processing
- **Documentation:** `apps/slate/CLAUDE.md`

### Shared Libraries

#### Fusain - Serial Protocol Library
- **Status:** ✅ Implemented
- **Location:** `modules/lib/fusain/`
- **Purpose:** Shared C library for Fusain serial protocol
- **Usage:** Used by both Helios (ICU) and Slate (controller)
- **Features:** CRC-16-CCITT, byte stuffing, packet encoding/decoding
- **Protocol Specification:** `origin/documentation/source/specifications/fusain/` (Sphinx docs)
- **Reference Implementations:**
  - **C:** `modules/lib/fusain/` - Embedded C for Zephyr RTOS
  - **Go:** `tools/heliostat/pkg/fusain/` - Reference Go implementation
- **Documentation:** `modules/lib/fusain/CLAUDE.md`

### Development Tools

#### Heliostat - Fusain Protocol Analyzer
- **Status:** ✅ Implemented
- **Location:** `tools/heliostat/`
- **Language:** Go
- **Purpose:** Real-time Fusain protocol analyzer and error detector
- **Features:** Cobra CLI, TUI mode, error detection, statistics tracking
- **Reference Go Implementation:** `pkg/fusain/` - Canonical Go implementation of Fusain protocol
- **Documentation:** `tools/heliostat/CLAUDE.md`

---

## Common Patterns

### Creating a New Firmware Application

1. **Ask questions first:**
   - What hardware platform? (RP2350A, RP2354A, other?)
   - What's the primary purpose?
   - Will it share code with existing projects?
   - What communication protocols are needed?

2. **Create directory structure:**
   ```
   apps/[firmware-name]/
   ├── CLAUDE.md              # Reference org CLAUDE.md in header
   ├── README.md
   ├── CMakeLists.txt
   ├── prj.conf               # Zephyr Kconfig
   ├── Taskfile.dist.yml      # Firmware-specific tasks
   ├── app.overlay            # Device tree
   ├── boards/                # Board-specific overlays
   ├── include/               # Public headers
   └── src/                   # Source files
   ```

3. **Reference existing patterns:**
   - Look at `apps/helios/` for structure examples
   - Follow Zephyr conventions
   - Use shared libraries from `modules/lib/` when applicable

### Creating a Shared Library

1. **Ask questions first:**
   - Which firmwares will use this library?
   - Should it be Zephyr-specific or portable?
   - Does it need hardware abstraction?

2. **Create in `modules/lib/`:**
   ```
   modules/lib/[library-name]/
   ├── CMakeLists.txt
   ├── Kconfig
   ├── include/               # Public API headers
   └── src/                   # Implementation
   ```

3. **Register in west manifest** (if needed)

---

## Safety & Best Practices

### Security
- Never commit secrets (.env files, credentials, keys)
- Validate all external inputs (UART, commands, sensor data)
- Be careful with command injection, buffer overflows

### Embedded Systems
- Consider memory constraints (limited RAM/flash on RP2350)
- Avoid dynamic allocation where possible
- Use static analysis tools
- Test on real hardware, not just in emulation

### Code Quality
- Avoid over-engineering - implement what's needed
- Don't add features beyond what's requested
- Keep solutions simple and focused
- Only add error handling for scenarios that can actually occur
- Don't create abstractions for one-time operations

---

## Resources

### Zephyr Documentation
- **Zephyr RTOS:** https://docs.zephyrproject.org/
- **West Workspace:** https://docs.zephyrproject.org/latest/develop/west/index.html
- **Device Tree:** https://docs.zephyrproject.org/latest/build/dts/index.html

### Hardware Documentation
- **RP2350 Datasheet:** https://pip-assets.raspberrypi.com/categories/1214-rp2350/documents/RP-008373-DS-2-rp2350-datasheet.pdf?disposition=inline
- **RP2350 Product Brief:** https://datasheets.raspberrypi.com/rp2350/rp2350-product-brief.pdf
- **RP2350 Hardware Design Guide:** https://datasheets.raspberrypi.com/rp2350/hardware-design-with-rp2350.pdf
- **Pico 2 Documentation:** https://www.raspberrypi.com/documentation/microcontrollers/pico-series.html

### Project Documentation

Each project has its own CLAUDE.md file with detailed documentation:

**Firmware Applications:**
- **Helios:** `apps/helios/CLAUDE.md` - Burner ICU firmware (RP2350A)
  - State machine, PID control, serial protocol, Zbus architecture
  - Controllers: temperature, motor, pump, glow plug
  - Complete operational documentation

- **Slate:** `apps/slate/CLAUDE.md` - Slate firmware (RP2350A)
  - LVGL display integration
  - Helios telemetry monitoring
  - Serial communication via Fusain library

**Shared Libraries:**
- **Fusain:** `modules/lib/fusain/CLAUDE.md` - Fusain serial protocol library (C)
  - Shared C library for Fusain protocol encoding/decoding
  - Used by both ICU (Helios) and controller (Slate)
  - CRC-16-CCITT, byte stuffing, packet framing
  - Message types, telemetry bundles, command structures

**Development Tools:**
- **Heliostat:** `tools/heliostat/CLAUDE.md` - Fusain protocol analyzer (Go)
  - Real-time packet decoder and validator
  - Error detection mode with statistics tracking
  - TUI interface with Bubbletea
  - **Reference Go implementation:** `pkg/fusain/` - Canonical Go implementation

**Documentation:**
- **Sphinx Docs:** `origin/documentation/CLAUDE.md` - Public documentation
  - Build with `task html`, preview with `task serve`
  - Zephyr-style theme with dark mode
  - reStructuredText source in `source/`
- **Fusain Specifications:** `origin/documentation/source/specifications/fusain/`
  - Protocol overview, packet format, message definitions
  - Implementation guide, communication patterns
  - Physical layer options (LIN, RS-485, TCP, WebSocket)

---

## Troubleshooting

### West Issues
- **"west not found":** Activate venv or check `.envrc`
- **Manifest errors:** Check `origin/west.yml` syntax
- **Module not found:** Run `west update`

### Build Issues
- **CMake errors:** Check `CMakeLists.txt` and module dependencies
- **Kconfig errors:** Verify `prj.conf` options are valid
- **Device tree errors:** Check overlay syntax and node references

---

## Contact & Support

This is the Thermoquad project organization. For questions or issues:
- Check project-specific CLAUDE.md files first
- Refer to Zephyr documentation for RTOS questions
- Ask clarifying questions before making assumptions

---

**Last Updated:** 2026-01-09

**Organization Maintainer:** kazw

---

## How AI Assistants Should Use This File

1. **Read this file first** when working on any Thermoquad project
2. **Then read the project-specific CLAUDE.md** for the firmware/library you're working on
3. **Ask questions** when conventions are unclear or requirements are ambiguous
4. **Follow the patterns** established in existing projects (especially Helios)
5. **Update this file** if you discover organization-wide patterns that should be documented

**Remember:** This organization values clarity and communication. When in doubt, ask!

---

## Content Integrity Check

A **content integrity check** verifies consistency across all CLAUDE.md files in the organization. This ensures documentation remains accurate and cross-references are valid.

**Scope:** Content integrity checks analyze **only the CLAUDE.md files themselves**. They do not analyze source code, configuration files, or other project files.

**What to Check:**
- Path references to protocol specifications and implementations
- Package and library names
- Project status descriptions
- Last Updated dates
- Cross-references between files
- Contradictions between files (e.g., conflicting specifications)
- Inconsistencies in terminology or naming conventions

**CLAUDE.md Files to Check:**
1. `origin/CLAUDE.md` - Organization-level (this file)
2. `origin/documentation/CLAUDE.md` - Sphinx documentation
3. `modules/lib/fusain/CLAUDE.md` - Fusain C library
4. `apps/helios/CLAUDE.md` - Helios ICU firmware
5. `apps/slate/CLAUDE.md` - Slate controller firmware
6. `tools/heliostat/CLAUDE.md` - Heliostat analyzer (Go)

**When to Run:**
- After updating protocol specifications
- After renaming packages or changing paths
- After updating project status
- Periodically to catch documentation drift

**How to Request:**
Ask the AI assistant to "run a content integrity check on all CLAUDE.md files"

---

## CLAUDE.md Reload

A **CLAUDE.md reload** reads all CLAUDE.md files in the organization to refresh the AI assistant's context with the latest documentation.

**Purpose:** Ensures the AI assistant has current information from all project documentation, especially useful when:
- Starting a new session after documentation updates
- Switching between projects and needing full context
- After another user or process has updated CLAUDE.md files
- When you want to ensure the assistant has the complete picture

**CLAUDE.md Files to Load:**
1. `origin/CLAUDE.md` - Organization-level (this file)
2. `origin/documentation/CLAUDE.md` - Sphinx documentation
3. `modules/lib/fusain/CLAUDE.md` - Fusain C library
4. `apps/helios/CLAUDE.md` - Helios ICU firmware
5. `apps/slate/CLAUDE.md` - Slate controller firmware
6. `tools/heliostat/CLAUDE.md` - Heliostat analyzer (Go)

**How to Request:**
Ask the AI assistant to "reload all CLAUDE.md files"
