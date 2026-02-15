# OpenClaw Compute Agent - Software Architecture

**Component:** openclaw-compute (Device Agent)  
**Version:** 1.0.0  
**Platform:** macOS 12.0+ (Apple Silicon)  
**Language:** Swift (UI), Rust (Core), Python (Runtime)  
**Date:** February 15, 2026

---

## Overview

The OpenClaw Compute Agent is a background daemon that turns idle Mac devices into distributed compute nodes. It monitors system idle state, communicates with the central scheduler, executes compute tasks in isolation, and reports results.

**Design Principles:**
- **Zero user impact:** Only runs when truly idle, pauses instantly on user activity
- **Security first:** Sandboxed execution, no network access for job code
- **Efficient:** Minimal resource overhead (< 50 MB RAM, < 1% CPU when idle)
- **Transparent:** Clear UI showing contribution stats and settings

---

## Architecture Overview

```
┌────────────────────────────────────────────────────────┐
│                  USER INTERFACE                        │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │
│  │  Menu Bar    │  │  Preferences │  │  Dashboard  │  │
│  │  (SwiftUI)   │  │  (SwiftUI)   │  │  (Web View) │  │
│  └──────────────┘  └──────────────┘  └─────────────┘  │
└────────────────────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│                 AGENT CORE (Rust)                      │
│  ┌──────────────────────────────────────────────────┐  │
│  │            State Machine / Orchestrator          │  │
│  └──────────────────────────────────────────────────┘  │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐  │
│  │   Idle      │  │  WebSocket  │  │   Task       │  │
│  │  Monitor    │  │   Client    │  │  Executor    │  │
│  └─────────────┘  └─────────────┘  └──────────────┘  │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐  │
│  │  Resource   │  │  P2P Data   │  │  Metrics     │  │
│  │  Monitor    │  │  Transfer   │  │  Collector   │  │
│  └─────────────┘  └─────────────┘  └──────────────┘  │
└────────────────────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│              EXECUTION LAYER (Python)                  │
│  ┌──────────────────────────────────────────────────┐  │
│  │    Sandbox Environment (Container/VM)            │  │
│  │  ┌────────────────────────────────────────────┐  │  │
│  │  │  User Job Code (train.py, render.py, etc) │  │  │
│  │  └────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

See full documentation at: https://github.com/dutchiono/openclaw-distributed-compute/blob/main/docs/agent-architecture.md

---

**Document Version:** 1.0.0  
**Last Updated:** February 15, 2026  
**Author:** OpenClaw Core Team
