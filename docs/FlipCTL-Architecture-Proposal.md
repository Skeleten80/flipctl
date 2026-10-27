# FlipCTL — Architecture Proposal & Documentation Outline

**Status:** Community proposal (open for feedback)  
**Related open task:** [Add the FlipCTL sub-project](https://github.com/flipperdevices/flipperone-docs/issues/404)  
**Repository:** https://github.com/flipperdevices/flipctl

---

## Overview

FlipCTL is a lightweight, daemon-based UI framework for embedded and headless Linux systems. It provides a navigation-friendly interface optimized for small screens (256×144 px and similar) without requiring a full desktop environment.

Originally developed for Flipper One, FlipCTL is designed to work on any Linux system — from single-board computers and routers to servers and headless appliances. The goal is simple installation (`apt install flipctl`) and immediate usefulness, whether the device has a built-in display or not.

**Core idea:** Wrap existing Linux tools and system services in a consistent, button-driven interface instead of forcing a full desktop onto a tiny screen.

---

## The Problem FlipCTL Solves

Desktop environments (GNOME, KDE, etc.) are designed for large, high-resolution displays and mouse/keyboard input. On a 256×144 screen they become unusable.

Traditional TUI libraries (ncurses, etc.) lack the pixel-level richness needed for clear status bars, icons, and structured menus on a small LCD. Companion mobile apps add complexity and require a second device.

FlipCTL solves this by providing:

- A pixel-aware but lightweight UI model
- Hardware-agnostic input (D-pad + soft keys, keyboard, or touch)
- Direct integration with system tools and services
- Multiple rendering targets from the same data model

---

## Vision

On any Linux system, FlipCTL should be immediately useful:

1. Install with a single package.
2. Connect a FlipCTL Control Panel (or use the built-in screen on Flipper One).
3. Get a working dashboard for system status, networking, and tools — no Xorg, no desktop, no configuration.

Long-term goals:

- Make adding a physical HMI to any embedded Linux device as easy as `apt install flipctl`.
- Provide a clean plugin model so wrapping existing CLI tools (nmap, ping, NetworkManager, etc.) is straightforward.
- Support multiple frontends (LCD, TUI, Web, Desktop) from shared UI logic.

---

## Architecture

FlipCTL is structured as a modular system with clear separation between system interaction, UI logic, and rendering.

```
┌────────────────────────────────────────────────────────┐
│                  Control Interfaces                     │
│  Flipper One LCD │ Control Panel │ TUI │ Web │ Desktop  │
└─────────────────────────┬───────────────────────────┘
                           │
┌─────────────────────────▼─────────────────────────────┐
│                 Frontend / Renderer                     │
│  HTML/JS or React + Renderer                            │
│  (headless WebKit/DRM, React+Ink TUI, browser, desktop) │
└─────────────────────────┬───────────────────────────┘
                           │ Plugin API / IPC
┌─────────────────────────▼───────────────────────────┐
│                      Core Daemon                        │
│  Input routing • Screen state • Plugin lifecycle       │
│  System integration (systemd, NetworkManager, etc.)    │
└─────────────────────────┬───────────────────────────┘
                           │
┌─────────────────────────▼───────────────────────────┐
│                   Plugins / Wrappers                    │
│  nmap • ping • traceroute • NetworkManager • disk •    │
│  power • custom services (any language)                │
└────────────────────────────────────────────────────────┘
```

### Key Design Principles

- **Separation of concerns**: Data and UI logic are independent of the renderer. The same plugin output can be displayed as a pixel UI, TUI, or web page.
- **Thin wrappers**: Plugins call existing tools rather than reimplementing them.
- **Hardware agnostic input**: The core sees abstract events (Up, Down, Left, Right, OK, Back, SoftKey1–5).
- **Daemon model**: Runs as a system service; frontends connect to it.
- **Minimal dependencies**: Prefer existing Linux infrastructure (systemd, NetworkManager, standard CLI tools).

---

## Core Components

### 1. Core Daemon

Responsibilities:

- Plugin discovery and lifecycle management
- Input event routing to the active plugin/screen
- Screen state management (current menu, focus, overlays)
- Communication with frontends
- System-level actions (power, service control) when not delegated to plugins

The daemon exposes a simple API (JSON-RPC over Unix socket or stdio for plugins).

### 2. Frontend / UI Logic

Currently envisioned as HTML + JavaScript (or React) for rapid development and component reuse. The frontend describes *what* to show and *how* to navigate; the renderer decides *how* to draw it.

### 3. Renderer

Multiple renderers share the same UI description:

| Renderer              | Target                          | Notes                                      |
|-----------------------|---------------------------------|--------------------------------------------|
| Headless WebKit + DRM | Flipper One / Control Panel LCD | Direct framebuffer, no X11/Wayland         |
| React + Ink           | Terminal / SSH                  | Character-based TUI                        |
| Standard browser      | Web UI                          | Same React codebase                        |
| Native desktop        | Development / workstation       | Optional                                   |

### 4. Plugins

Plugins are the extension mechanism. They wrap CLI tools or system APIs and expose structured data + actions to the frontend.

---

## Plugin System

A plugin is a process (or shared library) that:

1. Declares itself to the daemon (name, icon, menu entries, capabilities).
2. Receives input events and navigation commands.
3. Returns screen descriptions or result data.
4. Can be written in any language that can speak the protocol (Python, Go, Rust, shell, etc.).

### Proposed Plugin Protocol (Sketch)

Communication via JSON messages over stdio or a Unix socket.

**Registration example:**
```json
{
  "type": "register",
  "name": "nmap",
  "version": "0.1",
  "icon": "network",
  "entries": [
    {"id": "scan", "label": "Nmap Scan", "icon": "radar"}
  ]
}
```

**Screen description example (returned by plugin):**
```json
{
  "type": "screen",
  "title": "Nmap Scan",
  "items": [
    {"id": "target", "type": "input", "label": "Target", "value": ""},
    {"id": "type", "type": "select", "label": "Scan type", "options": ["Quick", "Full", "Ping"]},
    {"id": "start", "type": "action", "label": "Start Scan"}
  ],
  "softkeys": ["Back", "Help", "", "", "Start"]
}
```

**Result / progress example:**
```json
{
  "type": "update",
  "items": [
    {"id": "status", "label": "Scanning...", "progress": 0.42},
    {"id": "hosts", "type": "list", "items": ["192.168.1.1", "192.168.1.10"]}
  ]
}
```

This is intentionally simple so plugins stay thin and language-agnostic.

### Example Plugins (Priority)

1. **System Status** — CPU load, uptime, disk usage, memory, network interfaces, reboot/shutdown.
2. **Nmap** — Target input → scan type → progress → host/port list → details. (Addresses open UI task.)
3. **NetworkManager** — Wi-Fi scan/connect, Ethernet status, connection profiles.
4. **Ping / Traceroute** — Simple network diagnostics.
5. **Disk / Storage** — Mount points, usage, basic actions.

---

## Frontends & Control Interfaces

| Frontend                  | Description                                                                 | Priority |
|---------------------------|-----------------------------------------------------------------------------|----------|
| Flipper One               | Built-in 256×144 px LCD + physical controls. Includes boot menu.           | Primary  |
| FlipCTL Control Panel     | Standalone module with matching screen and buttons (USB or SPI).           | High     |
| TUI (Terminal UI)         | Usable over SSH or local console (React + Ink or equivalent).              | High     |
| Web UI                    | Browser-based, same layout logic.                                           | Medium   |
| Desktop App               | Native or Electron-style app for development and workstation use.          | Low      |

### Flipper One Physical Interface (Reference)

- **Screen:** 2.39″ diagonal, 256 × 144 px, 6-bit grayscale
- **Controls:** 5-button D-pad + soft keys (Escape, View, Power, Edit, Run) + optional PTT
- Navigation is menu-driven with clear focus and soft-key labels at the bottom of the screen

---

## Installation & Usage (Target Experience)

```bash
# On a Debian-based system
sudo apt update
sudo apt install flipctl

# Start the service (or it starts automatically)
sudo systemctl enable --now flipctl

# Connect a Control Panel via USB, or open the Web UI / TUI
```

On Flipper One the service is pre-installed and the built-in display is the default frontend.

---

## Contribution Guide

We welcome architecture proposals, plugin implementations, frontend work, and documentation.

### How to Contribute

1. **Read** this document and the current [FlipCTL page](https://docs.flipper.net/one/cpu-software/flipctl).
2. **Discuss** major changes in the relevant GitHub issue or repository discussion.
3. **Propose** concrete architecture or plugin designs via Pull Request to the `flipctl` repository.
4. **Implement** a minimal working prototype if possible (even a single plugin + one frontend is valuable).

A strong contribution includes:

- Clear description of components and how they interact
- Plugin system vision with at least one concrete example
- Minimal working code (or detailed interface specification)
- Consideration for the small-screen constraints

The author of a compelling, well-documented architecture proposal may be invited to help lead FlipCTL development.

### Open Related Tasks

- [Add the FlipCTL sub-project](https://github.com/flipperdevices/flipperone-docs/issues/404) (docs)
- [UI for Nmap application](https://github.com/flipperdevices/flipperone-ui/issues/15)
- [Linux Mode Home Screen UI concept](https://github.com/flipperdevices/flipperone-ui/issues/27)
- [Describe operating modes + draw scheme](https://github.com/flipperdevices/flipperone-ui/issues/1)

---

## Proposed Next Steps

1. Finalize and document the plugin protocol (schema + example messages).
2. Implement a minimal core daemon that can load one plugin and route input.
3. Create two reference frontends: simple Web UI + TUI.
4. Ship two reference plugins: System Status and Nmap.
5. Add this content (or a refined version) as the top-level FlipCTL section in the Developer Portal.
6. Iterate based on community feedback and real hardware testing.

---

## References

- FlipCTL documentation page: https://docs.flipper.net/one/cpu-software/flipctl
- FlipCTL repository: https://github.com/flipperdevices/flipctl
- Flipper One Developer Portal: https://docs.flipper.net/one
- Open tasks: https://docs.flipper.net/one/open-tasks

---

*This document is a community proposal intended to accelerate FlipCTL development. Feedback, corrections, and alternative designs are welcome.*
