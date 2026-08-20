---
name: "esp32-lvgl-port-en"
description: "Ports LVGL components, display drivers, and input drivers to ESP32/ESP-IDF projects, generates configuration and main dependencies, and fixes build compatibility. Invoke when users request ESP LVGL porting, display integration, or LVGL build fixes."
---

# ESP32 LVGL Porting Assistant

## Scope

This Skill handles ESP-family chips and ESP-IDF projects. Its main purpose is to reliably integrate LVGL, displays, touch panels, and external buttons into an ESP-IDF project. Use C for firmware code, English for runtime logs, and English comments in this English Skill. Never use `goto`; prefer `switch-case` for state and event handling.

Supported devices include, but are not limited to: ESP32, ESP32-S2, ESP32-S3, ESP32-C3, ESP32-C6, ESP32-H2, and ESP32-P4. Supported ESP-IDF versions include 4.4 and 5.x.

## Mandatory Principles

1. **Ask before changing files.** Do not guess the LVGL version, chip, display controller, or driver. Do not install components or modify the project until the required parameters are known.
2. **Scan the existing project first.** Check `idf_component.yml`, `main/CMakeLists.txt`, `CMakeLists.txt`, `sdkconfig`, `lv_conf.h`, display/touch drivers, and existing LVGL tasks to avoid duplicate porting work.
3. **The LVGL version must be explicit.** Identify it from component manifests, `lv_version.h`, `lv_conf.h`, or lock files. If it cannot be identified, ask the user. LVGL 8.x and 9.x APIs must not be mixed.
4. **Do not damage user code.** Read relevant files before editing and prefer small, focused changes. If an existing display driver, input driver, or LVGL task can be reused, explain the reuse plan instead of creating another one.
5. **All LVGL object operations must run inside the existing LVGL task or under the proper LVGL lock.** Business tasks, driver callbacks, and ISRs must not directly call `lv_obj_*`, `lv_label_*`, or similar LVGL APIs.
6. **Do not invent component versions or driver capabilities.** Verify external component information through the component registry, project metadata, or actual source code.

## Step 1: Collect Information Through Conversation

Do not list every question in the first reply or use a form-like message such as “Before I start the port, please provide the following information.” Talk with the user like an engineer: ask only about the most important current topic in each turn, then choose the next question from the user's answer.

### Conversation Rules

1. **Check what the user already provided.** Extract known values from the user's description, the currently open project, and files already read. Repeat known values only for confirmation; do not ask them again.
2. **Start with LVGL and the chip.** For example: “Let me confirm the basic environment first: are you using LVGL 8 or 9, which exact ESP chip is it, and is PSRAM available?”
3. **Ask about one topic per turn.** Do not combine LVGL, chip, display, touch, and project status into a numbered checklist. If the user answers several topics at once, record them and skip those questions.
4. **Ask natural follow-up questions.** If the user says “The display is ST7789,” ask for its resolution, bus, and pins next instead of asking for the display model again.
5. **Clarify uncertainty before proceeding.** Do not guess the LVGL patch version, PSRAM, display pins, or driver source. You may offer common choices, but the user must confirm them.
6. **Summarize only after the required information is collected.** Briefly list the LVGL version, chip/PSRAM, IDF, display, driver, input device, and project state, then ask whether the summary is correct. Do not edit files, install components, or generate code until the user confirms.
7. **Do not force a one-shot form.** If the user does not know a value, explain how to find it in `idf_component.yml`, `sdkconfig`, the schematic, driver files, or with `idf.py --version`.

### Usual Information Order

Adapt the order to the conversation. Usually confirm:

- LVGL major and full version, such as `8.3.11` or `9.2.2`;
- ESP chip model and PSRAM availability;
- ESP-IDF version;
- Display controller, resolution, bus (SPI/I2C/RGB/8080), and important pins;
- Display driver source: bundled component driver, existing driver, or user-provided files/repository;
- Touch controller and bus, or external buttons/encoder;
- Whether the input device is registered with LVGL or used only by business logic;
- Whether the project already has LVGL, a display task, display/input drivers, and whether `idf.py build` passes.

### Conversation Example

```text
Assistant: Let me confirm the basic environment first: are you using LVGL 8 or 9, which exact ESP chip is it, and is PSRAM available?
User: LVGL 9, ESP32-S3, with PSRAM.
Assistant: Got it. I recorded LVGL 9 and ESP32-S3 with PSRAM. Do you know the full LVGL version and ESP-IDF version? If not, I can show you where to check them.
User: The display is ST7789.
Assistant: Thanks, the display controller is ST7789. What are its resolution, bus type (SPI or another interface), and CS, DC, RST, SCLK, and MOSI pins?
```

Do not replace this conversational flow with a fixed “1, 2, 3…” question list.

## Step 2: Scan the Project and Confirm the Plan

After receiving the answers, perform these checks in order:

1. Locate the project root and read the project-level `CMakeLists.txt`, `main/CMakeLists.txt`, `main/idf_component.yml` if present, `sdkconfig`, and `sdkconfig.defaults`.
2. Search for `lvgl.h`, `lv_conf.h`, `lv_version.h`, `esp_lvgl_port`, `lvgl_port`, `lv_task_handler`, `lv_timer_handler`, `esp_lcd`, `esp_lcd_panel`, `touch`, `button`, and `encoder`.
3. Classify the project:
   - LVGL and a display driver already exist: only add missing configuration or an adapter layer.
   - A display low-level driver exists but LVGL does not: reuse the low-level driver and add LVGL.
   - The project is empty: generate the smallest buildable porting skeleton.
   - A complete UI exists: do not overwrite it; only connect the component and initialization entry point.
4. Present the scan result, files to be changed, component versions, driver source, and input-device integration method in English. Wait for user confirmation before writing files.

## Step 3: Find a Suitable LVGL Component

Always check the Espressif Component Registry first:

`https://components.espressif.com/components?q=`

Search and verify candidates using the confirmed LVGL version, ESP-IDF version, and chip model:

- Component name, release version, and dependencies;
- Support for the target ESP-IDF major version;
- Whether it includes an `lv_conf.h` example, `esp_lvgl_port`, or display adapter;
- Compatibility with `esp_lcd`, touch components, and the target display controller.

If the search result is insufficient, consult the matching LVGL documentation or component source. Always show the actual link and explain why the component was selected. Do not automatically select an unverified version just because the search is incomplete.

## Step 4: Generate the Component and Configuration

After the user confirms the plan, immediately perform the actual implementation. Do not merely say "completed", "will create", or list planned files. Use the project root `<project_root>` located in Step 2: the display driver must be placed in `<project_root>/components/`, never in `<project_root>/main/`, and never at a hard-coded machine-specific path. If no suitable local component exists, create one according to the current ESP-IDF rules.

### Mandatory Step 4 Execution Protocol

1. Check whether `<project_root>/components/` exists; if it does not, actually create it and the component directory.
2. Use file tools to actually create or edit the component `CMakeLists.txt`, `include/*.h`, `.c`, version-matched `lv_conf.h`, and `LVGL_CONFIG_GUIDE.md`; always read an existing file before editing it.
3. After each batch of file changes, read the files again or list their directory to verify they exist, have the correct location, and contain the changes.
4. Do not claim that LVGL porting code is integrated until writing and read-back verification have both completed.
5. If file creation/editing permission is unavailable, tool access is denied, or the project path does not exist, state the exact blocker and request access or the path. Do not skip Step 4 and fabricate a completion result.
6. Once Step 4 verification passes, continue directly with Steps 5, 6, and 7. Do not stop at a step boundary unless confirmed hardware information is missing or a real build error occurs.

Use a structure like this; choose the component name according to its actual function:

```text
components/
└── display_driver/           # Standalone display-driver component
    ├── CMakeLists.txt
    ├── include/
    │   ├── display_driver.h   # Pins, macros, public types, and declarations
    │   └── lvgl_demo.h        # Demo macros, enums, and entry declaration with Chinese comments
    ├── display_driver.c      # Function definitions and driver implementation
    ├── lvgl_demo.c           # Demo implementation
    ├── lv_conf.h             # Copy from the actual LVGL source/component and configure by version
    └── LVGL_CONFIG_GUIDE.md  # UTF-8 configuration and usage guide
```

Component file rules:

- **Mandatory IO macros:** Define every GPIO and SPI/I2C/RGB/8080 bus pin, backlight, reset, chip-select, and touch-interrupt IO number with a clearly named `#define` macro. Do not use bare numeric pin values in `.c` files, function calls, or structure initializers.
- **Mandatory header declarations:** Put every public function declaration, all macro definitions, and all enumerations in the component `.h` file under `include/`. Do not leave any of them only in a `.c` file.
- **Mandatory Chinese comments:** Every function declaration, macro definition, and enumeration in `.h` files must have an accurate Chinese comment explaining its purpose, unit, valid range, or calling constraint. Runtime logs must remain in English.
- Put display-driver function definitions, state machines, and hardware operations in the component `.c` file;
- Put pin definitions, macros, structs, enums, and public function declarations in the `.h` file under the component's `include/` directory;
- Keep `main/` for the application entry point, business logic, and component calls. Do not copy or reimplement the display driver there;
- If an existing component already contains the display driver, reuse and modify it instead of writing a duplicate driver in `main/`;
- If no component exists, create `components/<component_name>/CMakeLists.txt`, `include/<component_name>.h`, and the corresponding `.c` file, with correct `REQUIRES` / `PRIV_REQUIRES` declarations;
- Component directories and filenames must follow the current ESP-IDF component rules, not temporary directory conventions.
- **Mandatory componentized demo:** Create the public Demo header at `components/<component_name>/include/lvgl_demo.h` and implement the Demo at `components/<component_name>/lvgl_demo.c`. Do not place Demo implementation directly in `main/`.
- `lvgl_demo.h` must contain all Demo-required includes, LCD resolution/draw-buffer/pixel-clock macros, all LCD GPIO macros, Demo enums if any, and the Demo entry-function declaration. All macros, enums, and declarations require Chinese comments. GPIO macros must be semantic; bare numeric pin values are prohibited.
- `lvgl_demo.c` must contain private UI creation functions such as `create_demo_screen()`, Demo initialization, panel registration, LVGL display registration, and UI creation under the LVGL lock. It must not define `app_main()`.
- `main`'s `app_main()` must contain only necessary application startup logic and include `lvgl_demo.h` to call the Demo entry function, such as `lvgl_demo_start()`. Do not duplicate LCD, LVGL, or UI initialization in `app_main()`.
- Add `lvgl_demo.c` to the component `CMakeLists.txt` `SRCS`. After writing, read back `lvgl_demo.h`, `lvgl_demo.c`, the `main` entry, and `CMakeLists.txt` to verify that the entry is actually called, declarations match, and the Demo source will be built.

### LVGL Configuration Rules

- Copy the configuration file matching the actual LVGL version from the project's LVGL source or resolved LVGL component into the display component. Do not invent a version-mismatched `lv_conf.h`;
- `lv_conf.h` must match the selected LVGL version;
- Resolution, color depth, memory, fonts, logging, and draw buffers must reflect the display and PSRAM configuration;
- Use LVGL/component defaults for uncertain options instead of enabling many features based on assumptions;
- Configuration comments and generated documentation must use the selected documentation language; runtime logs must remain in English.

Always generate an `LVGL_CONFIG_GUIDE.md` usage guide alongside the component. Use Simplified Chinese UTF-8 by default; use English only when the user explicitly requests it. Do not mix Chinese and English in the same guide.

The guide must explain in plain language:

- Where the component and `lv_conf.h` are located;
- How to change display resolution, color depth, draw buffers, fonts, and feature switches;
- What to check first when RAM is insufficient, the display is corrupted, or touch coordinates are reversed;
- Which LVGL 8.x and 9.x APIs must not be mixed;
- How to clean and rebuild after changing the configuration.

Unless explicitly requested, do not create extra README files or generic documentation.

## Step 5: Generate the Main Component `.yml` File

Create or update `main/idf_component.yml`. The file extension must be `.yml`, not `.yaml`, and the file must use the required ESP-IDF Component Manager filename. It must be a valid Component Manager manifest, not an arbitrary YAML file. Add dependencies based on the confirmed versions, for example:

```yaml
dependencies:
  idf: ">=5.1"
  lvgl/lvgl: "~9.2.2"
  espressif/esp_lvgl_port: "^2.4.0"
```

Rules:

- Add only dependencies that are actually used; do not add unused display or touch components;
- Version constraints must match the confirmed LVGL major version and IDF compatibility;
- Do not disguise a local component as a remote dependency;
- Preserve other existing YAML settings and dependencies.

If the project uses an older ESP-IDF or Component Manager that does not support the current manifest syntax, explain the limitation and use the configuration method supported by that project.

## Step 6: Connect the Display and Input Devices

### Display Path

Implement the path according to the actual driver type:

```text
app_main()
  -> initialize the bus
  -> initialize esp_lcd or the confirmed display driver
  -> initialize LVGL
  -> create the display object and flush callback
  -> create/start the LVGL task
  -> periodically call lv_timer_handler() or the handler required by that version
```

- Display registration and refresh APIs must match LVGL 8.x or 9.x;
- SPI, RGB, and I2C drivers must not be substituted for one another;
- Bind tasks to a core only when needed on dual-core chips; do not use dual-core-specific architecture on single-core chips;
- Validate handler period, task stack, and buffer size against actual memory.

### Touch, Buttons, and Encoder

Implement according to the user's choice:

- If registered with LVGL, create the correct LVGL input device and implement the read callback, coordinate range, key mapping, and event state;
- If used only by business logic, the driver should report events to a queue or state machine, and business tasks must not operate LVGL objects directly;
- Define touch rotation, mirroring, calibration, and screen-boundary handling;
- In an ISR, only perform the minimum event handoff; never call LVGL APIs;
- Handle input events with `switch-case`; never use `goto`.

## Step 7: Compatibility Fixes and Build Loop

After porting, always perform this loop:

1. Run `idf.py build` in the current project. Do not claim completion based only on static analysis.
2. If the build fails, inspect the first real error and classify it as component resolution, header, API version, CMake dependency, configuration, link, memory, or driver-interface related.
3. Make a focused fix and build again. Do not hide the error with unrelated changes.
4. Repeat “read error -> locate source -> edit -> build” until the build passes, or clearly record the external driver or hardware information still required from the user.
5. Do not pass the build by deleting functionality, commenting out real errors, inventing functions, forcibly disabling warnings, or mixing LVGL APIs.
6. After a successful build, verify that component resolution, LVGL version, target chip, and input configuration match the confirmed plan.
7. **Mandatory second execution after a successful build:** Treat this as invoking this Skill again and immediately perform “LVGL configuration and documentation archival”. Do not end after reporting a successful build.
   - Locate the configuration template or configuration file from the LVGL component source actually resolved by this build: look for `lv_conf.h` first, then the version-provided `lv_conf_template.h` if needed. Do not invent a file or copy one from a different LVGL major version.
   - Copy it to the display component as `<project_root>/components/<component_name>/lv_conf.h`, which is the project's single LVGL configuration file. When the source is `lv_conf_template.h`, the copied destination must still be named `lv_conf.h`.
   - Actually edit this component-local `lv_conf.h` for the confirmed display, color depth, draw buffers, and PSRAM configuration; retain version defaults for unknown settings.
   - Actually create or update `LVGL_CONFIG_GUIDE.md` in the same component directory, in Simplified Chinese UTF-8. It must explain the source and path of `lv_conf.h`, common configuration items, how to rebuild after edits, and configuration incompatibilities between LVGL 8.x and 9.x.
   - Read back `<component_name>/lv_conf.h` and `LVGL_CONFIG_GUIDE.md` to verify they exist and contain valid content. Then run `idf.py build` again to verify the project still builds after the configuration file is copied.
8. Do not provide a final completion summary until the second build succeeds or the actual error caused by the copied configuration has been clearly reported.

Runtime logs must be in English, for example:

```c
ESP_LOGI(TAG, "LVGL port initialized");
ESP_LOGE(TAG, "Display driver initialization failed");
```

## Completion Checklist

- [ ] The full LVGL version was asked for and confirmed
- [ ] ESP chip, PSRAM, and ESP-IDF version were confirmed
- [ ] Display controller, resolution, bus, pins, and driver source were confirmed
- [ ] Touch/buttons/encoder and LVGL registration choice were confirmed
- [ ] The existing project was scanned to avoid duplicate components and LVGL tasks
- [ ] The Component Registry was checked and the selection reason was explained
- [ ] `lv_conf.h` was placed in the appropriate component and matches the LVGL version
- [ ] The configuration guide was generated in the user's language as UTF-8
- [ ] `main/idf_component.yml` was generated or updated with actual dependencies
- [ ] All LVGL APIs run in the correct task or under the correct lock
- [ ] No `goto` is used; states and events prefer `switch-case`
- [ ] `idf.py build` was executed
- [ ] Build failures were fixed, or remaining external blockers were clearly listed
- [ ] The final summary lists changed files, component versions, build result, and any manual wiring/calibration still required

## Trigger Examples

- “Port ST7789 to LVGL 9 on an ESP32-S3.”
- “Add LVGL and touch support to an ESP-IDF project.”
- “LVGL does not compile; fix it according to the IDF version.”
- “Generate the LVGL configuration file and main component dependencies.”
