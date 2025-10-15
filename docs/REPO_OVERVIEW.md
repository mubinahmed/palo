# Palo Server Repository Overview

This document summarizes the structure of the Palo 5.1 MOLAP server source tree and highlights the main responsibilities of each area.

## Build and Runtime Basics

The project is built with CMake and depends on common C++ toolchains (Visual Studio on Windows or GCC on Linux) together with third-party libraries such as Boost, OpenSSL, ICU, zlib, and Google Performance Tools for profiling. The top-level README also documents the basic CMake workflow for generating build files and compiling the server, along with simple launch instructions for the `palo` binary.【F:README†L1-L55】

Runtime behavior is driven by command line arguments and configuration parsed through `PaloOptions`. This component exposes switches for HTTP listeners, data directories, authentication controls, session TTL, drill-through, and more, ensuring a consistent interface between CLI flags and the `palo.ini` configuration file.【F:Programs/PaloOptions.h†L31-L160】【F:palo.ini.sample†L5-L200】

## Executable Entry Points (`Programs/`)

The `Programs` directory hosts the server entry points. `palo.cpp` wires together configuration parsing, logging, HTTP interfaces, and extension loading. It installs platform-specific signal handlers, registers crash dumps, loads optional modules, and ultimately launches the HTTP interface that blocks on `select()` until shutdown. GPU-specific workers can also be registered when compiled with the proper feature flag.【F:Programs/palo.cpp†L35-L200】

`PaloHttpInterface` encapsulates the HTTP/HTTPS listeners used by the server. It instantiates admin and public listeners based on configuration, coordinates TLS when enabled, and maintains the event loop over the active sockets. This layer throws an error if no listeners can be initialized, ensuring that the service fails fast when misconfigured.【F:Programs/PaloHttpInterface.cpp†L32-L158】

Supporting sources in this directory cover data loading (`PaloLoader`), autosave timers, command-line option parsing, GPU worker stubs, and Windows service wrappers, illustrating how the executable coordinates the broader library components.

## Core Libraries (`Library/`)

The `Library` tree contains the reusable subsystems that implement OLAP functionality.

* **OLAP Model (`Library/Olap`)** – Defines the domain model for databases, cubes, dimensions, rules, sessions, and system metadata. The `Server` class aggregates these pieces, handles licensing, login workflows, drill-through control, encryption settings, and tracks active sessions. It inherits from `Commitable`, exposing persistence hooks used during saves and shutdowns.【F:Library/Olap/Server.h†L37-L199】
* **Calculation Engine (`Library/Engine`)** – Implements query execution, aggregation, and rule processing. `EngineBase` defines plan node structures, splash modes for value distribution, caching, and asynchronous execution primitives shared across CPU and GPU implementations.【F:Library/Engine/EngineBase.h†L34-L199】
* **Infrastructure Packages** – Additional subdirectories deliver logging, threading, networking, HTTP/S servers, dump handlers, schedulers, worker orchestration, and utility collections. These components allow the server to manage I/O, concurrency, task scheduling, and diagnostic output consistently across the codebase.

## HTTP API and Browser Assets (`Api/`)

The `Api` directory bundles HTTP templates, JavaScript assets, and API definition files. The `*.api` descriptors enumerate REST endpoints for server management, database operations, cube manipulation, element management, and rule handling; each entry documents the path, purpose, and required authentication token. HTML templates and static assets back the bundled administrative web interface and documentation browser.【F:Api/api_overview.api†L1-L80】

## Configuration and Resources

* `palo.ini.sample` serves as the canonical configuration reference, documenting how CLI toggles interact with the configuration file, HTTP listener definitions, logging destinations, worker integration, and feature-specific thresholds such as splashing or goal-seek limits.【F:palo.ini.sample†L5-L200】
* The `Resource` directory (along with `PaloDocumentation`) contains localized strings, help content, and other runtime assets consumed by the HTTP UI and documentation engine.
* `Modules/CMake` and `Config/*.cmake` provide CMake helper modules and generated header templates that control feature switches during the build.

Together, these layers produce a modular MOLAP server: the executable orchestrates startup and networking, `Library` houses the OLAP engine and supporting infrastructure, and the `Api` plus configuration directories deliver administrative tooling and customization hooks.
