---
name: xrobot-workflow
description: Build and maintain XRobot/LibXR embedded modules, including Python CLI-driven module generation, dependency/configuration setup, generated entry points, and lifecycle-safe C++ module implementations.
---

# XRobot Workflow

Use this skill when a request concerns the `libxr` or `xrobot` Python packages, XRobot module repositories, `Modules/modules.yaml`, `User/xrobot.yaml`, generated `xrobot_main.hpp`, or implementation of a LibXR/XRobot C++ module.

## Operating rules

1. Inspect the existing workspace and its platform/RTOS before changing code. Preserve established naming, CMake, and generated-file conventions.
2. Use the installed CLI as the source of truth. Check `--help` when flags are uncertain; do not invent commands or YAML keys. See [references/cli-and-module-contract.md](references/cli-and-module-contract.md).
3. Treat `Modules/` repositories and `User/xrobot_main.hpp` as generated or synchronized outputs. Change source configuration or module headers first, then regenerate. Never hand-edit constructor arguments in generated C++; regenerate it from the source of truth.
4. Keep module metadata in the `/* === MODULE MANIFEST V2 === */` block. Make constructor arguments, template arguments, required hardware, and dependencies parseable and consistent with the actual C++ constructor.
5. Implement modules around constructor-time initialization plus periodic `OnMonitor()`/`Monitor()` work as required by the local API. The constructor must acquire/validate required hardware, establish invariants, and leave the object usable or fail immediately through the repository's established mechanism (such as an exception, assertion/panic, or validated factory). Do not add a deferred `Init()`/lazy-initialization phase. An explicit `start()` may be provided only for intentional post-construction activation (for example, beginning I/O or a task); it must not hide required initialization, and its state/error behavior must be explicit and idempotent according to the local convention.
6. Prefer declaring module-specific structs, enums, aliases, and `constexpr` values inside the module class. Use nested names such as `Config` and `Detail` (`Module::Config`, `Module::Detail`) instead of namespace-level names such as `ModuleConfig` or `ModuleDetail`; keep a type outside only when it is genuinely shared independently of that class.
7. Follow LibXR constraints: no destructor-based shutdown for normal module lifetime, no runtime allocation/free in ISR or hot paths, no blocking in callbacks/interrupts, explicit thread-vs-ISR context, deterministic completion behavior for every I/O operation, and no platform SDK types in public cross-platform interfaces.
8. Separate the main data stream from `OnMonitor()`. For IRQ/DMA/UART/CAN-driven inputs, use the hardware callback/ISR only for bounded acknowledgement, buffer exchange, and state transitions; use preallocated lock-free or bounded handoff structures where the producer/consumer contract requires them. Perform parsing, transformation, and publication in a worker/application path, not by polling the stream from `OnMonitor()`.
9. Treat `OnMonitor()` as the periodic application-level hook dispatched by `ApplicationManager`: use it for health/staleness checks, diagnostics, rate/frequency checks, fault handling, and reading a coherent snapshot. It normally runs in task/application context, not ISR context, and is not the primary high-rate data path. Keep it bounded so one module does not stall all modules.
10. Require an alert/data-ready/event IRQ path for the module's main data stream. Do not fall back to polling in `OnMonitor()`, even when an event path is inconvenient or absent; if the required event/IRQ capability is unavailable, reject the design or fail fast during construction through the repository's established mechanism.
11. After edits, run the narrowest useful validation: parse the manifest, regenerate the main entry, configure/build or run repository tests, and report unavailable toolchain or hardware validation.

## Typical workflow

- New workspace/module: install `xrobot` and `libxr` in one environment, run `xrobot_setup`, create or synchronize modules, inspect the generated manifest, then implement the module.
- Existing module change: read its manifest and constructor, update the header/source and dependency metadata, run `xrobot_mod_parser`, then regenerate with `xrobot_gen_main`.
- New instance/dependency: update `Modules/modules.yaml` for repositories and `User/xrobot.yaml` for instances/constructor values; use `xrobot_add_mod` rather than editing generated entry code.

## Module argument source of truth

- To change an existing module instance's constructor values, edit that instance in `User/xrobot.yaml`, then run `xrobot_gen_main`. Do not change the arguments directly in `User/xrobot_main.hpp`.
- To change the module's constructor signature or argument schema, update the module header/C++ implementation and its `MODULE MANIFEST` (`constructor_args`, `template_args`, hardware, and dependencies). Then re-run the module parser and regenerate the YAML/entry point as appropriate.
- Treat `User/xrobot.yaml` as the source of truth for per-instance values, the module header/MANIFEST as the source of truth for the module interface, and `User/xrobot_main.hpp` as generated output only.

## Review checklist

- Does every module constructor argument appear in the manifest and generated configuration?
- Are all dependencies explicit and resolvable, with no version conflict?
- Is `XRobotMain()` regenerated after configuration changes?
- Can any callback, ISR, or `OnMonitor()` path block, allocate, free, recurse unexpectedly, or touch platform-specific public types?
- Does construction establish a fully valid object or fail immediately, with no required `Init()` call or hidden lazy setup?
- If `start()` exists, is it only explicit activation after successful construction, with clear state/error behavior?
- Is ownership clear without requiring a destructor, and are all I/O operations bound to a completion mode?
