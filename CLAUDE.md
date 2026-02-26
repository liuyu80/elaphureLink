# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

```powershell
# Restore NuGet packages
msbuild elaphureLink.sln -t:restore -p:RestorePackagesConfig=true

# Build Debug version
msbuild elaphureLink.sln -target:elaphureLinkWpf\elaphureLink_Wpf -m /property:Configuration=Debug

# Build Release version
msbuild elaphureLink.sln -target:elaphureLinkWpf\elaphureLink_Wpf -m /property:Configuration=Release

# Clean build artifacts
rm bin\* -vb -Recurse -Force -Include *.exp, *.idb, *.ilk, *.iobj, *.ipdb, *.lastbuildstate, *.lib, *.obj, *.res, *.tlog
```

Requirements: Visual Studio 2019+, Windows x64, .NET Framework 4.7.2

## Architecture Overview

elaphureLink is a CMSIS-DAP compliant debugger solution for Arm Keil µVision IDE. The system has 4 main components:

### Component Layers

```
µVision IDE
    │
    ▼
┌─────────────────────────────────────────┐
│  elaphureLinkAGDI (AGDI层)              │  ← DbgCM.cpp, PDSCDebug.cpp
│  - Implements µVision AGDI interface    │
│  - Dynamic DLL loading of RDDI          │
└──────────────┬──────────────────────────┘
               │ DLL function pointers (rddi_dll.cpp)
               ▼
┌─────────────────────────────────────────┐
│  elaphureLinkRDDI (RDDI层 - DLL)       │  ← rddi_dap.cpp, dap_jtag.cpp
│  - Implements RDDI/DAP interface        │
│  - Shared memory IPC to Proxy           │
└──────────────┬──────────────────────────┘
               │ Shared Memory + Windows Events
               ▼
┌─────────────────────────────────────────┐
│  elaphureLinkProxy (Proxy层)            │  ← protocol.cpp, SocketClient.hpp
│  - TCP/UDP communication with devices   │
│  - Exports DAP device                    │
└──────────────┬──────────────────────────┘
               │ TCP/USB (CMSIS-DAP)
               ▼
┌─────────────────────────────────────────┐
│  Server Providers (Debug Hardware)      │  ← Custom DAP implementations
└─────────────────────────────────────────┘
```

### Key IPC Mechanism

The RDDI-to-Proxy communication uses shared memory with Windows Events (producer-consumer pattern):

- **Shared memory**: `common/ipc_common.hpp` defines `el_memory_t` structure
  - `producer_page`: commands sent from RDDI
  - `consumer_page`: responses from Proxy
  - `info_page`: device capabilities and status

- **Synchronization**: `k_producer_event` and `k_consumer_event` Windows events

### Important Source Files

| Layer | Key Files | Purpose |
|-------|-----------|---------|
| AGDI | `elaphureLinkAGDI/DbgCM/rddi_dll.cpp` | RDDI function pointer table |
| AGDI | `elaphureLinkAGDI/DbgCM/DbgCM.cpp` | Main AGDI implementation |
| RDDI | `elaphureLinkRDDI/rddi_dap.cpp` | RDDI/DAP function implementations |
| RDDI | `elaphureLinkRDDI/ElaphureLinkRDDIContext.cpp` | Debug session state |
| Proxy | `elaphureLinkProxy/protocol.cpp` | TCP/UDP protocol handler |
| Proxy | `elaphureLinkProxy/proxy.cpp` | Proxy core logic |
| Common | `common/ipc_common.hpp` | Shared memory structures |
| Common | `common/rddi_dap.h`, `rddi_dap_cmsis.h` | CMSIS-DAP definitions |

### Protocol Documentation

See `docs/proxy_protocol.md` for the elaphureLink proxy protocol (CMSIS-DAP compliant, handshake + data transfer phases).

See `docs/proxy_api.md` for Proxy API reference (el_proxy_init, el_proxy_start_with_address, etc.).
