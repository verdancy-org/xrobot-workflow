# XRobot CLI and Module Contract

Use this reference when exact commands, files, or metadata are needed. XRobot documentation: <https://xrobot.work>.

## Installation

`libxr` is the C++ library; `xrobot` and the CodeGenerator are Python packages. Use one installation mode consistently:

```sh
python -m pip install libxr xrobot
# or, in an isolated CLI environment:
pipx install libxr xrobot
```

Do not mix `pip` and `pipx` installations of the same package in one shell environment.

## Workspace files and commands

- `Modules/modules.yaml`: repository/module list, for example `xrobot-org/BlinkLED` or `your-namespace/YourModule@dev`.
- `Modules/sources.yaml`: optional module-source index/mirrors.
- `User/xrobot.yaml`: module instances, IDs, constructor values, template values, and global settings.
- `User/xrobot_main.hpp`: generated C++ entry point; never use it as the configuration source.

### Argument/configuration source of truth

- Change a module instance's constructor values in `User/xrobot.yaml`, then regenerate with `xrobot_gen_main`. Never edit those values directly in generated `User/xrobot_main.hpp`.
- Change a module's constructor signature or argument schema in the module header/C++ implementation and its `/* === MODULE MANIFEST V2 === */` block (`constructor_args`, `template_args`, `required_hardware`, and `depends`), then run `xrobot_mod_parser` and regenerate the YAML/entry point as appropriate.
- The YAML owns per-instance values; the module header/MANIFEST owns the module interface; generated C++ is output only.

Core commands:

```sh
xrobot_setup
xrobot_init_mod --config Modules/modules.yaml --directory Modules --sources Modules/sources.yaml
xrobot_create_mod MySensor --desc "IMU interface module" --hw i2c1
xrobot_mod_parser --path ./Modules/MySensor/
xrobot_add_mod your-namespace/YourModule@main
xrobot_add_mod MySensor --instance-id sensor0
xrobot_gen_main --output User/xrobot_main.hpp
xrobot_gen_main --config User/xrobot.yaml --output User/xrobot_main.hpp --hw hw
```

For an STM32CubeMX CMake project, the LibXR CodeGenerator entry point is:

```sh
xr_cubemx_cfg -d .
```

It discovers the `.ioc`, updates/initializes LibXR, parses the project, generates `User/app_main.cpp`, and updates CMake integration. The underlying tools are `xr_parse_ioc -d . -o ./.config.yaml`, `xr_gen_code_stm32 -i ./.config.yaml -o ./User/app_main.cpp`, and `xr_stm32_cmake .`. Optional `xr_cubemx_cfg` flags include `-t <terminal>`, `--xrobot`, `--commit`, `--git-source`, and `--git-mirrors`.

`xrobot_setup` initializes missing config, recursively fetches modules, regenerates the entry point, and refreshes `Modules/CMakeLists.txt`. `xrobot_init_mod` also detects dependency/version conflicts; resolve those in repository configuration.

Generated `User/app_main.cpp` has `User Code Begin/End` regions intended for custom additions; keep user code inside those regions so regeneration preserves it. The generated `app_main()` is expected not to return during normal operation.

## Manifest

Every discoverable module header should contain a parseable block like:

```cpp
/* === MODULE MANIFEST V2 ===
module_description : IMU interface module
constructor_args : [ ]
template_args : [ ]
required_hardware :
  - i2c1
depends : [ ]
=== END MANIFEST === */
```

The manifest drives parser output, dependency traversal, generated YAML, and constructor calls. Keep names/types/defaults aligned with the real constructor. `xrobot_create_mod` can scaffold constructor and template arguments (`--constructor kp=1.0 --template T=float`) and dependencies (`--depends MySensor`); inspect the generated header before implementing behavior.

## Generated entry semantics

`xrobot_gen_main` emits an `XRobotMain(LibXR::HardwareContainer& hw)`-style function, creates an `ApplicationManager`, statically constructs module instances, calls `appmgr.MonitorAll()`, and sleeps according to `global_settings.monitor_sleep_ms`. Module-instance references such as `'@BlinkLED_0'` can be used in `User/xrobot.yaml` when a constructor needs another instance or member.

## Implementation contract

### `OnMonitor()` versus the main data stream

`ApplicationManager::MonitorAll()` periodically dispatches each module's `OnMonitor()` from application/task context. Review official `xrobot-org` modules with this division in mind:

- High-rate sensor and communications modules receive bytes/frames or data-ready events through IRQ/callback/DMA paths and often finish parsing/publishing in a dedicated worker thread. The callback/ISR should do bounded handoff and state advancement only.
- `OnMonitor()` is normally a supervision hook: inspect the latest coherent snapshot, detect stale links or invalid values, report diagnostics, check frequency/rate, and handle faults. It is not the primary high-rate ingestion path and should not poll an IRQ-driven stream.
- Require an alert/data-ready/event IRQ path for the module's main data stream. Do not fall back to polling in `OnMonitor()`; if the required event/IRQ capability is unavailable, reject the design or fail fast during construction through the repository's established mechanism.
- If monitor code reads state accumulated by worker/ISR paths, use the module's concurrency contract to obtain a coherent snapshot. Task-context synchronization is acceptable where appropriate; APIs documented as task-only must not be called from ISR.
- Use lock-free or bounded preallocated queues/rings/double buffers for IRQ-to-thread handoff when needed by the data rate and ownership model. Do not call every `OnMonitor()` implementation lock-free, and do not assume every device must be IRQ-driven.

- Perform all required initialization in the constructor: validate dependencies and hardware, allocate or bind durable resources, establish invariants, and either produce a usable object or fail immediately using the repository's established mechanism. Do not require a separate `Init()` method, delayed initialization, or hidden lazy setup.
- A `start()` method is optional and must mean explicit activation after successful construction (for example, start I/O, timers, or a worker). It is not a substitute for constructor initialization; document its state/error contract and make repeated calls safe according to the project convention.
- Do not add a destructor merely to make normal module operation work. Avoid destruction-dependent ownership and shutdown semantics; if a platform resource truly requires release, document the exceptional lifecycle and keep it outside ISR/hot paths.
- Treat callbacks and ISRs as non-blocking: no sleeps, mutex waits, heap allocation/free, unbounded loops, logging with unknown cost, or large application workflows.
- Make context explicit where the API exposes `in_isr` or callback-safe variants. Use the ISR for bounded hardware/buffer handoff and defer application logic to a thread/monitor path.
- Bind each I/O operation to a clear completion mode (callback, blocking wait, polling, or explicit ignore/NONE) using the LibXR `Operation` model.
- Keep public module interfaces platform-neutral. Hide HAL handles, RTOS task types, and SDK-specific types in platform adapters.
- Prefer static/preallocated storage, bounded queues, and explicit state transitions. If dynamic allocation is unavoidable, perform it at a controlled initialization/phase boundary with a documented upper bound.
