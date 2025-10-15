# Palo Source Code Overview

## Project Purpose and Build Basics
Palo is an open-source MOLAP (Multidimensional Online Analytical Processing) server distributed by Jedox. The repository ships the server code alongside build instructions that rely on CMake and toolchains such as GCC on Linux or Visual Studio on Windows.【F:README†L1-L55】

## High-Level Runtime Flow
The main executable is defined in `Programs/palo.cpp`. Startup logic parses command-line options, loads optional extension modules, instantiates the OLAP server, and wires the job analyser that maps HTTP requests onto backend jobs. It then loads persisted data, opens HTTP/HTTPS listeners, and blocks in the server loop until shutdown. During shutdown it flushes data to disk and tears down the job system and extensions.【F:Programs/palo.cpp†L35-L330】

Supporting binaries such as `palorun` are thin wrappers that load the server DLL on Windows before calling the shared `PaloMain` entry point.【F:Programs/palorun.cpp†L1-L40】

## Configuration and Options
Runtime configuration is handled by `PaloOptions`, which encapsulates command-line switches and init-file parameters for ports, data directories, persistence behavior, security, GPU usage, and more. The options object can parse both CLI arguments and configuration files before being used to update global server state.【F:Programs/PaloOptions.h†L31-L200】

A sample `palo.ini` documents every configurable flag, including automatic database loading/committing, HTTP/admin listeners, logging, and advanced features like goal-seek limits or CSV persistence toggles.【F:palo.ini.sample†L1-L158】

## Data Loading Lifecycle
`PaloLoader` coordinates loading server metadata and cube data from the file system. It encapsulates auto-load/auto-add behavior, persists changes when auto-commit is enabled, and finalizes startup by preparing the session context.【F:Programs/PaloLoader.h†L40-L88】

## Core OLAP Engine
The OLAP engine lives under `Library/Olap`. `Server` manages global state (sessions, encryption, GPU activation, blocking mode) and exposes persistence hooks to load/save databases. It also coordinates shutdown, maintains login state, and offers the global mutex used when committing data.【F:Library/Olap/Server.h†L85-L233】

Each `Database` aggregates dimensions and cubes, tracks load status, and knows how to persist itself. Databases cooperate with the journaling subsystem for crash recovery and enforce naming rules.【F:Library/Olap/Database.h†L99-L157】 Individual `Cube` objects own the multidimensional cell storage, track load state, and configure behaviors such as splash limits, goal-seek constraints, and caching policies.【F:Library/Olap/Cube.h†L64-L155】

## HTTP Front-End and Dispatcher
`PaloHttpInterface` bootstraps one or more HTTP/HTTPS listeners, optionally with dedicated admin ports. It multiplexes client sockets, creates connection tasks, and coordinates shutdown by committing outstanding work. GPU-specific maintenance hooks live here when that feature is enabled.【F:Programs/PaloHttpInterface.cpp†L32-L239】

Concrete request handling is implemented in `PaloHttpServer`, a specialization of the generic `HttpServer` infrastructure. It wires the Palo-specific command handlers, exposes the optional web-based API browser, and registers documentation routes derived from the `Api` templates.【F:Library/PaloHttpServer/PaloHttpServer.h†L39-L100】

Incoming requests are turned into jobs by `PaloJobAnalyser`, which extends the generic dispatcher framework. It maps job names to concrete Palo job factories so requests can execute asynchronously via the worker pool.【F:Library/PaloDispatcher/PaloJobAnalyser.h†L30-L73】

## Background Workers and Autosave
`WorkersCreator` spins up background worker threads on startup and coordinates their lifetime during shutdown or restarts, ensuring long-running tasks have time to finish. The autosave timer created in `palo.cpp` works alongside this infrastructure to persist server state periodically.【F:Library/Worker/WorkersCreator.h†L33-L59】【F:Programs/palo.cpp†L228-L299】

## Extensibility and Modules
During boot the server scans the extensions directory for dynamically loaded modules. Extensions can override the HTTP interface, HTTPS bindings, or even provide custom server factories and job analysers before the core runtime starts listening for requests.【F:Programs/palo.cpp†L192-L215】

This architectural layering—options parsing, data loading, OLAP engine, HTTP front-end, dispatcher, and background workers—provides a clear separation of concerns while keeping the MOLAP core decoupled from transport and scheduling concerns.
