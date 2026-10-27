# STM32 workflow: CubeMX, CMake, GCC and VS Code

<p>
  <img alt="Platforms" src="https://img.shields.io/badge/platforms-Linux%20%7C%20Windows-blue">
  <img alt="Toolchain" src="https://img.shields.io/badge/toolchain-STM32CubeCLT%20%2B%20CMake-informational">
  <a href="LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-green"></a>
</p>

How to set up every new STM32 project so that it follows one consistent,
reproducible style — written so you can work through it without prior knowledge.

The projects are fully platform-neutral: a project created on Windows builds
unchanged on Linux and vice versa, because the whole build lives in versioned text
files (CMakeLists, presets, toolchain file), not in IDE settings. Where the steps
differ, this guide shows **Linux (Fedora)** and **Windows** side by side.

> **Shell commands:** `cmake`, `git`, `gh` and `STM32_Programmer_CLI` are typed the
> same way in a Linux terminal and in Windows PowerShell. Paths are written with `/`;
> on Windows `\` works as well.

---

## Contents

- [Glossary: our tools and why](#glossary-our-tools-and-why)
- [VS Code in brief: layout and shortcuts](#vs-code-in-brief-layout-and-shortcuts)
  - [Output, Debug Console, Terminal — who shows what?](#output-debug-console-terminal--who-shows-what)
- [Part 1: Set up the computer (once per machine)](#part-1-set-up-the-computer-once-per-machine)
  - [1.1 Install STM32CubeCLT (compiler, flasher, debugger)](#11-install-stm32cubeclt-compiler-flasher-debugger)
  - [1.2 Install CubeMX (hardware configurator)](#12-install-cubemx-hardware-configurator)
  - [1.3 Base tools and VS Code](#13-base-tools-and-vs-code)
  - [1.4 Connect Git and GitHub once](#14-connect-git-and-github-once)
- [Part 2: Create a new project (per project)](#part-2-create-a-new-project-per-project)
  - [2.1 Generate it in CubeMX](#21-generate-it-in-cubemx)
  - [2.2 First build (terminal or VS Code, identical result)](#22-first-build-terminal-or-vs-code-identical-result)
  - [2.3 Output files: .hex and .bin (and optionally .dfu)](#23-output-files-hex-and-bin-and-optionally-dfu)
  - [2.4 Project hygiene (our standard)](#24-project-hygiene-our-standard)
  - [2.5 Git and GitHub — what we do and why](#25-git-and-github--what-we-do-and-why)
- [Part 3: The daily work cycle](#part-3-the-daily-work-cycle)
- [Side note: FreeRTOS — via the CubeMX checkbox or by hand?](#side-note-freertos--via-the-cubemx-checkbox-or-by-hand)
- [Side note: peripheral init as separate .c/.h files per peripheral](#side-note-peripheral-init-as-separate-ch-files-per-peripheral)
- [Part 4: Checklist for every new project](#part-4-checklist-for-every-new-project)
- [Part 5: Troubleshooting — typical errors and how to fix them](#part-5-troubleshooting--typical-errors-and-how-to-fix-them)

---

## Glossary: our tools and why

| Tool | What it is | Why we use it |
|---|---|---|
| **STM32CubeMX** | ST's graphical hardware configurator and code generator | Turns clicks (pins, clock, peripherals) into the complete project skeleton — correct and in minutes instead of hours of manual work. It NEVER compiles anything itself. |
| **STM32CubeCLT** | ST's toolbox: compiler (arm-none-eabi-gcc), flasher (STM32_Programmer_CLI), debug server (ST-LINK_gdbserver); on Windows including CMake and Ninja | One fixed, versioned installation instead of drifting system packages → builds that are exactly reproducible in five years. Free. |
| **arm-none-eabi-gcc** | The actual C compiler for ARM controllers (part of CubeCLT) | Industry standard, free, open source — no IAR/Keil licences needed. |
| **CMake + Ninja** | Build system: CMake describes WHAT is built, Ninja runs it fast | The build lives in versioned text files instead of IDE settings → identical result on every machine, whether from the terminal or VS Code. |
| **VS Code** | Editor with extensions for STM32, debugging and CMake | Free, cross-platform; replaces CubeIDE/IAR as the everyday workspace. |
| **Git** | Version control: records every change to the project locally | Every state can be restored, every change can be traced ("who/when/why"). |
| **GitHub** | Online home for Git projects | Backup, team exchange, the central "truth" — what is not on GitHub does not officially exist. |

How it fits together in one sentence: **CubeMX generates the project, CMake+GCC
build it, VS Code is the workbench, Git/GitHub are the memory.** CubeMX is only
needed when creating the project and for hardware changes — the daily work (code,
build, flash, debug, commit) runs without it.

---

## VS Code in brief: layout and shortcuts

For anyone new to VS Code, four areas are enough to find your way: the **side bar**
on the left (icons for files, search, Git, debug, extensions), the **editor** in the
middle, the **panel** at the bottom (terminal, output, problems) and the **status
bar** at the very bottom (CMake preset and build button live there). The single most
important tool is the **Command Palette**: Ctrl+Shift+P opens a search field from
which EVERY command can be reached — whenever this guide says "Developer: Reload
Window" or "CMake: ...", that is what is meant: open the palette, type the command,
press Enter.

| Shortcut | Effect |
|---|---|
| **Ctrl+Shift+P** | Command Palette — the master key to everything |
| Ctrl+P | Quickly open a file in the project (type its name) |
| Ctrl+S | Save (before every build! unsaved = old state) |
| Ctrl+Shift+E | Side bar: file explorer |
| Ctrl+Shift+F | Side bar: search the whole project |
| Ctrl+Shift+X | Side bar: extensions |
| **Ctrl+Shift+D** | Side bar: Run & Debug (dropdown + F5) |
| Ctrl+\` (Ctrl+Ö on a German keyboard) | Panel: integrated terminal on/off |
| Ctrl+Shift+U | Panel: Output (build/CMake messages) |
| Ctrl+, | Settings |
| **F7** | Build (CMake Tools; same as the build button in the status bar) |
| **F5** | Debug: starts the configuration selected in the dropdown |
| F10 / F11 | In the debugger: step over / step into |
| Shift+F5 | End the debug session (with "Attach": only disconnect) |

The two keys people mix up: **F7 builds, F5 flashes and debugs** — and F5 always
uses the configuration selected at the top of the Run & Debug view (Ctrl+Shift+D).
Do NOT use the play/bug icons at the bottom in the status bar (see Troubleshooting:
they try to run the firmware on the PC).

---

### Output, Debug Console, Terminal — who shows what?

The panel at the bottom has three tabs that beginners constantly confuse, because
all three "show text". The distinction is easy once you ask: WHO is talking here?

- **Output (Ctrl+Shift+U)** — the **extensions** talk here. Read only, nothing to
  type. The dropdown at the top right selects the channel (CMake/Build, the ST
  extension, …). *Example:* after F7 you see the compiler calls here and, at the end,
  the memory table `FLASH: … B` — that is the "CMake/Build" channel.
- **Debug Console** — the **debugger** talks here during an F5 session, and you can
  talk BACK to it: the input line at the bottom evaluates expressions. *Example:*
  while the program is stopped at a breakpoint, type `ir_sensor_edge_count` and press
  Enter — the console shows the variable's current value. (The Cortex-Debug start
  messages and the `-cp` path from Troubleshooting also appear here.)
- **Terminal (Ctrl+\`)** — a **real shell** in your project folder, identical to an
  external terminal window. Here you type commands yourself. *Example:*
  `cmake --build --preset Release` or `git status`. In addition, Cortex-Debug creates
  a **"gdb-server"** tab here during a debug session — it holds the output of
  ST-LINK_gdbserver, the place to read WHY a "GDB Server Quit Unexpectedly" happened
  (e.g. "no ST-Link found").

Memory aid: **Output = log of the tools, Debug Console = dialogue with the debugger,
Terminal = dialogue with the system.**

---

## Part 1: Set up the computer (once per machine)

### 1.1 Install STM32CubeCLT (compiler, flasher, debugger)

From st.com → search "STM32CubeCLT" → Get Software (an ST account or e-mail is
required).

**Linux (Fedora):** download the *Generic Linux Installer*, then:

```bash
cd ~/Downloads
unzip stm32cubeclt_*.sh.zip
chmod +x stm32cubeclt_*.sh          # ls shows the exact file name
sudo sh ./stm32cubeclt_*.sh
```

Accept the licence with `y`, keep the install location
`/opt/st/stm32cubeclt_<version>/`, and answer yes to the udev rules for the ST-Link
(otherwise only root may access the debugger). Then **log out and back in once** —
only then are the tools on the PATH.

**Windows:** download the *Windows installer* and run it; keep the default location
`C:\ST\STM32CubeCLT_<version>\`. The installer sets up the ST-Link USB driver and
adds all tools to the PATH. **CubeCLT for Windows includes CMake and Ninja** — no
extra installation needed. Then open a **NEW** PowerShell window (PATH changes only
apply to newly opened windows!).

Check — all three must answer:

```bash
arm-none-eabi-gcc --version
STM32_Programmer_CLI --version
ST-LINK_gdbserver --version
```

The installed version (it is in the install path) belongs in every project's README
— that is the reproducibility record. Several versions can be installed side by side.

### 1.2 Install CubeMX (hardware configurator)

From st.com → "STM32CubeMX". **At least version 6.11** — older ones cannot generate
CMake.

- **Linux:** download the Linux version, unpack it, run
  `./SetupSTM32CubeMX-<version>` (graphical installer).
- **Windows:** run the Windows installer.

### 1.3 Base tools and VS Code

**Linux (Fedora):**

```bash
sudo dnf install cmake ninja-build git gh
```

Install VS Code as an **RPM from the Microsoft repository** — **NOT as a Flatpak**
from GNOME Software/Discover! The Flatpak sandbox blocks access to the toolchain under
/opt/st and to the ST-Link (learned the hard way):

```bash
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo tee /etc/yum.repos.d/vscode.repo << 'EOF'
[code]
name=Visual Studio Code
baseurl=https://packages.microsoft.com/yumrepos/vscode
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc
EOF
sudo dnf install code
```

**Windows** (CMake and Ninja already came with CubeCLT):

```powershell
winget install Git.Git
winget install GitHub.cli
winget install Microsoft.VisualStudioCode
```

(Alternatively the installers from git-scm.com and code.visualstudio.com; keep the
defaults in the Git installer.) Then open a NEW PowerShell window.

**Extensions** (in VS Code: Ctrl+Shift+X, search, Install):
**STM32Cube for Visual Studio Code** (publisher STMicroelectronics), **Cortex-Debug**,
**C/C++**, **CMake Tools**. The ST extension finds CubeCLT on its own; if it asks on
Windows, point it to `C:\ST\STM32CubeCLT_<version>`. For projects with the
`.vscode/extensions.json` described below, VS Code suggests the extensions
automatically when the folder is opened — one click on "Install". If no prompt
appears: type `@recommended` into the search field of the Extensions view and install
them there in one go.

### 1.4 Connect Git and GitHub once

```bash
git config --global user.name  "First Last"
git config --global user.email "address-of-your@github.account"
gh auth login        # GitHub.com -> HTTPS -> "Login with a web browser"
```

**Windows only**, additionally:

```powershell
git config --global core.autocrlf input
```

Windows ends text lines with CRLF, Linux with LF. This setting makes sure the
repository always contains LF — otherwise every switch between platforms produces
huge fake changes in `git diff`.

The e-mail should be the one of your GitHub account, so commits are linked to it.
`gh auth login` signs you in once via the browser — after that `git push` works
permanently without a password prompt. (Your normal GitHub password never works for
pushing — GitHub requires tokens, and `gh` manages them for you.)

---

## Part 2: Create a new project (per project)

### 2.1 Generate it in CubeMX

1. Start CubeMX → *File → New Project* → find the controller in the MCU selector
   (e.g. STM32G071CBTx) → double-click / *Start Project*.
2. Tab **Pinout & Configuration**: assign pins by clicking, enable peripherals and
   middleware (e.g. FreeRTOS). Tab **Clock Configuration**: set the clock.
3. Tab **Project Manager** — the three decisive fields:
   - **Project Name:** short, WITHOUT spaces or umlauts (e.g. `Project123`) — it also
     becomes the name of the build target.
   - **Project Location:** e.g. `~/projects/` or `C:\projects\` — the path also without
     spaces or umlauts (saves trouble with build tools).
   - **Toolchain / IDE: `CMake`** ← THIS is the core of the whole style. If it says
     CubeIDE or EWARM, you get the wrong project format.
4. Tab **Code Generator** (also in Project Manager): tick
   **"Generate peripheral initialization as a pair of '.c/.h' files per peripheral"** —
   every peripheral gets its own file instead of everything in main.c (reasons in the
   side note further down).
5. Click **GENERATE CODE**.

The result is the familiar structure: `.ioc` (the hardware configuration as a file),
`CMakeLists.txt`, `CMakePresets.json`, `cmake/gcc-arm-none-eabi.cmake`,
`cmake/stm32cubemx/CMakeLists.txt`, `Core/` (your workspace), `Drivers/` (ST
libraries), startup file and linker script.

### 2.2 First build (terminal or VS Code, identical result)

```bash
cd ~/projects/Project123      # Windows: cd C:\projects\Project123
cmake --preset Debug          # once: configure the build folder
cmake --build --preset Debug  # build
```

Or in VS Code: `code .`, accept the extension recommendation; if asked for a
"kit"/compiler, choose **"[Unspecified]"** (the preset selects the compiler itself!),
choose the preset "Debug", press F7.

The success criterion is always the memory table at the end:

```
Memory region   Used Size  Region Size  %age Used
         RAM:      xxxx B        xx KB     x.xx%
       FLASH:      xxxx B       xxx KB     x.xx%
```

Note these numbers in the README — they are the reference every later build and every
other machine is measured against. The finished program is in
`build/Debug/<Name>.elf` (+ `.hex`/`.bin`).

#### Why two presets? Debug and Release are two different programs

The presets differ not only in their output folder but in the compiler switches
(set in `cmake/gcc-arm-none-eabi.cmake`): **Debug** builds with `-O0 -g3` — no
optimisation, full debug information. Every C line maps 1:1 to machine code, which is
why single-stepping, breakpoints and the variable view work cleanly. **Release**
builds with `-Os` — the compiler optimises for size (a much smaller, faster binary),
but it is hardly debuggable any more, and sloppy code (missing `volatile`, timing by
empty loops) suddenly behaves differently because the optimiser rearranges or removes
it. Hence the rule: **develop and hunt bugs in Debug — validate and ship ALWAYS the
Release.** Whatever was only tested in Debug is untested, because a different machine
program runs on the production device.

The Debug preset also defines the macro `DEBUG` (see `$<$<CONFIG:Debug>:DEBUG>` in
`cmake/stm32cubemx/CMakeLists.txt`). This lets you add test output that is guaranteed
to be absent from the production build — for example (assumes a UART enabled in
CubeMX):

```c
/* USER CODE BEGIN PD */
#ifdef DEBUG
  // Only present in the Debug build - empty in Release
  #define DBG_MSG(txt)  HAL_UART_Transmit(&huart2, (uint8_t *)(txt), sizeof(txt) - 1, 100)
#else
  #define DBG_MSG(txt)
#endif
/* USER CODE END PD */
```

Call it anywhere in the code:

```c
DBG_MSG("Alarm triggered\r\n");
```

In the Debug build the line appears on the UART (115200 8N1; read it on Linux e.g.
with `screen /dev/ttyUSB0 115200` from the `screen` package — quit with Ctrl+A, then K;
on Windows e.g. with PuTTY on the COM port of the USB-UART adapter). In the Release
build exactly the same source line produces **zero bytes of code** — no run time, no
flash usage, no accidental chatter on the production device.

### 2.3 Output files: .hex and .bin (and optionally .dfu)

CMake projects freshly generated by CubeMX produce **only the `.elf`** when built.
For flashing with CubeProgrammer, however, `.hex` is the standard. That is why EVERY
new project gets this block once — **at the end of the `CMakeLists.txt` in the project's
main folder** (NOT in the one under `cmake/stm32cubemx/`, which CubeMX overwrites on
every Generate; the root file is created by CubeMX only the very first time and never
touched again — additions there survive regeneration):

```cmake
# After linking: create .hex and .bin for CubeProgrammer
add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O ihex   $<TARGET_FILE:${CMAKE_PROJECT_NAME}> ${CMAKE_PROJECT_NAME}.hex
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${CMAKE_PROJECT_NAME}> ${CMAKE_PROJECT_NAME}.bin
)
```

What it does: `objcopy` (part of the GCC toolchain) converts the .elf into both formats
after every link. After adding it, build once from scratch
(`cmake --build --preset Debug --clean-first`, since the step only runs when linking)
— from then on every build leaves `<Name>.elf`, `.hex`, `.bin` and `.map` in
`build/Debug/` or `build/Release/`.

**Optional: .dfu for firmware updates in the field over USB.** Requirement: the
controller has USB hardware (e.g. STM32G0B1, F4, H5 — an STM32G071 has NO USB; there the
ROM bootloader runs over UART instead). Two ways:

1. **Without a .dfu file** — CubeProgrammer flashes the normal .hex directly through
   the DFU bootloader (put the device into the bootloader with BOOT0 first):

   `STM32_Programmer_CLI -c port=USB1 -w build/Release/Project123.hex`

2. **A real .dfu file** — needed when end users flash with a DfuSe update tool. It is
   created from the .hex with the script `dfuse-pack.py` from the dfu-util project
   (`0483:df11` is the USB ID of the ST bootloader):

```bash
# Linux:
sudo dnf install dfu-util                  # flash tool; the script dfuse-pack.py
                                           # is in the source repo of the dfu-util project
python3 dfuse-pack.py -i build/Release/Project123.hex -D 0x0483:0xdf11 Project123.dfu
```

```powershell
# Windows: install Python once, then download dfuse-pack.py from the dfu-util source repo
winget install Python.Python.3.12
python dfuse-pack.py -i build/Release/Project123.hex -D 0x0483:0xdf11 Project123.dfu
```

   Graphical alternative on Windows: the **DFU File Manager** from the ST package
   STSW-STM32080 (load .hex → Generate → .dfu).

If you want the .dfu automatically on every build, add the call as another `COMMAND`
line to the objcopy block above.

### 2.4 Project hygiene (our standard)

Four more files belong in every project. Either copy them from an existing project — or
create them with exactly the content below. `Project123` is the example project name
throughout.

> **Windows note:** the forward slashes `/` in the JSON paths are correct — VS Code
> understands them on Windows too; do not enter backslashes.

#### `.gitignore` (in the project's main folder)

Tells Git which files NOT to version. Basic rule: everything that can be recreated from
the sources at any time (build output), and pure tool or OS artefacts, does not belong
in the repository. `.gitignore` allows comments with `#`:

```gitignore
# ==== Build output - can be recreated from the sources at any time ====
build/
*.elf
*.hex
*.bin
*.map
*.dfu

# ==== Editor and tool caches ====
.cache/
.vscode/ipch
compile_commands.json

# ==== CubeMX working files and backups ====
.mxproject
*.bak

# ==== OS artefacts (they show up sooner or later) ====
.DS_Store               # macOS Finder
Thumbs.db               # Windows Explorer
desktop.ini             # Windows Explorer

# ==== In case colleagues use other editors/IDEs ====
*.swp                   # vim swap files
*~                      # backup files of various editors
.idea/                  # JetBrains IDEs (CLion etc.)
```

Good to know for later: a leading `!` takes a single file back out of an exclusion rule
(e.g. `!reference.hex`) — but the better place for shipped binaries is a GitHub release
(section 2.5), not the repository.

#### `.vscode/extensions.json`

The project's extension recommendations. When someone opens the project in VS Code,
"Do you want to install the recommended extensions?" appears automatically — one click
and the machine is ready to work. If the prompt does not appear (e.g. because
everything is already installed or it was dismissed earlier): open the Extensions view
(Ctrl+Shift+X) and type `@recommended` into the search field — the section "Workspace
Recommendations" shows the project's recommendations, and the cloud icon next to it
installs all missing ones at once. The IDs follow the pattern `publisher.extensionname`.

Worth knowing: VS Code reads its configuration files as **"JSON with comments"** —
`//` comments are allowed here (and in launch.json), and we use them deliberately as
documentation. In "normal" JSON (e.g. data for a program) comments are forbidden — do
not be surprised by errors elsewhere.

```jsonc
{
    "recommendations": [
        // Required for this workflow:
        "stmicroelectronics.stm32-vscode-extension",    // ST: finds CubeCLT, imports projects
        "marus25.cortex-debug",                         // debugging via ST-Link + GDB server
        "ms-vscode.cpptools",                           // C/C++ basics (see note below)
        "ms-vscode.cmake-tools"                         // CMake: presets, build button, F7

        // Optional but proven - to enable, uncomment the line
        // (and add a comma to the previous one!):
        // "ms-vscode.hexeditor",                       // view .bin and memory as a hex dump
        // "eamodio.gitlens",                           // Git history in the editor: who/when/why per line
        // "dan-c-underwood.arm"                        // syntax colours for ARM assembler (startup file)
    ]
}
```

How they interact: the ST extension brings its own IntelliSense engine
(stm32-cube-clangd). If VS Code reports a conflict with the C/C++ extension, disable
Microsoft IntelliSense (details in Troubleshooting, Part 5) — through
compile_commands.json, clangd knows exactly the flags of the real build.

#### `.vscode/launch.json`

The debug configuration — it turns F5 into the complete sequence "flash, start the GDB
server, stop at main()". We create TWO configurations right away; the second is worth
its weight in gold in everyday work:

```jsonc
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug (ST-Link)",          // display name in the Run & Debug dropdown
            "type": "cortex-debug",             // the Cortex-Debug extension
            "request": "launch",                // launch = flash, reset, restart
            "servertype": "stlink",             // GDB server from CubeCLT (ST-LINK_gdbserver)
            "cwd": "${workspaceFolder}",
            "executable": "${workspaceFolder}/build/Debug/Project123.elf",
            "device": "STM32G071CB",            // controller - see explanation below
            "interface": "swd",                 // SWD is the STM32 standard (alternative: jtag)
            "runToEntryPoint": "main",          // run to main() after start and stop there
            "svdFile": "",                      // path to an .svd file -> peripheral registers in the debugger
            "preLaunchTask": ""                 // e.g. a build task -> build automatically before F5
            // ,"rtos": "FreeRTOS"              // for FreeRTOS projects: task list in the debugger
            // ,"showDevDebugOutput": "raw"     // for connection problems: show all GDB messages
        },
        {
            // Attaches to the RUNNING device: nothing is flashed, nothing
            // is reset. Ideal for examining a fault that is happening right
            // now without losing it to a restart.
            "name": "Attach (no flashing)",
            "type": "cortex-debug",
            "request": "attach",                // attach instead of launch = only attach
            "servertype": "stlink",
            "cwd": "${workspaceFolder}",
            "executable": "${workspaceFolder}/build/Debug/Project123.elf",
            "device": "STM32G071CB"
        }

        // Optional third configuration for the special case "the fault only
        // happens in Release" - to enable, uncomment it and keep the comma
        // at the start (explanation below):
        // ,{
        //     "name": "Debug the Release build (optimised!)",
        //     "type": "cortex-debug",
        //     "request": "launch",
        //     "servertype": "stlink",
        //     "cwd": "${workspaceFolder}",
        //     "executable": "${workspaceFolder}/build/Release/Project123.elf",
        //     "device": "STM32G071CB"
        // }
    ]
}
```

The most important fields and how much leeway they have:

- **`device`** — how exact must the name be? With `servertype: stlink` the field is
  tolerant: the ST GDB server detects the connected chip itself, `STM32G071CB` and
  `STM32G071CBT6` both work (convention: without package/temperature suffix). BUT: if
  you ever use a SEGGER J-Link (`servertype: jlink`), the name must be written EXACTLY
  as in the SEGGER device database — there the field is strict.
- **`svdFile`** — an .svd file describes all peripheral registers of the controller;
  with a path entered, the debugger shows every register live under "XPeripherals"
  (e.g. timer counter, GPIO states) — extremely helpful for hardware debugging. CubeCLT
  ships the files, e.g.
  `/opt/st/stm32cubeclt_<version>/STMicroelectronics_CMSIS_SVD/STM32G071.svd` or
  `C:/ST/STM32CubeCLT_<version>/STMicroelectronics_CMSIS_SVD/STM32G071.svd`.
- **`executable` with "Attach"** — must match the firmware that is RUNNING on the
  device right now. Background: nothing is flashed on attach. The debugger only loads
  the "map" from the .elf on disk — which function and which source line sits at which
  flash address — and lays it over what is actually in the chip. If map and terrain do
  not match, the debugger shows nonsense. **Example:** flashed and debugged with F5 in
  the morning. In the afternoon main.c changed and was REBUILT with F7 — but not
  flashed. The device still runs the morning state, the .elf on disk is already the
  afternoon state. An attach then claims e.g. "stopped at line 120, LED routine" while
  the CPU is really somewhere in the ADC code, and variables show random values — none
  of this is a defect, just an outdated map. Rule of thumb: only attach if nothing was
  rebuilt since the last flash; when in doubt, flash once with F5 and it matches again.
  The same applies to devices in the field: the tagged state v1.0 belongs with the .elf
  from exactly that build — another reason to archive release states including their
  files as a GitHub release.
- **`preLaunchTask`** — refers to a task in `.vscode/tasks.json`; with it, F5 builds
  automatically before flashing. Leave it empty at first — building deliberately with F7
  prevents accidentally debugging an outdated state and wondering why.
- `${workspaceFolder}` is a VS Code variable for the project folder and stays exactly
  as written.

**Adjust per project:** `executable` (project name of the .elf, in all configurations)
and `device`; for FreeRTOS enable the `rtos` line.

**Does it need its own Release configuration?** Not for everyday work — that is why the
template only includes it commented out. Normally you debug the Debug build; the Release
configuration is for the rare but important special case "the fault ONLY happens in
Release" (typical with optimisation-sensitive code, e.g. a missing `volatile`). Two
things to know: first, our Release builds with `-g0`, i.e. WITHOUT debug symbols — the
debugger would be almost blind. Remedy: in `cmake/gcc-arm-none-eabi.cmake` replace `-g0`
with `-g3` in the Release branch. That is safe, because `-g` only changes the extra
information INSIDE the .elf file — the generated machine program, and with it the
flashed .hex, stays byte-identical. Second, Release debugging stays bumpy even with
symbols (the optimiser reorders: apparent jumping while stepping, variables showing
`<optimized out>`) — that is normal and not a defect. And to be clear: launch.json
contributes nothing to BUILDING and FLASHING a release — that always goes through the
terminal (`cmake --preset Release`, build, `STM32_Programmer_CLI`; the complete sequence
is in the walkthrough in Part 3). F5 and launch.json are purely debugging tools.

#### `README.md`

The project's business card — GitHub shows it as the start page. Rule: a new colleague
must be able to build, flash and place the state with the README alone. A proven
skeleton to fill in:

````markdown
# Project123 — <one line: what is the device/firmware?>

- MCU: <e.g. STM32G071CBT6 (Cortex-M0+, 128 KB flash, 36 KB RAM)>
- Clock: <e.g. 16 MHz HSI>
- RTOS: <none / FreeRTOS (CMSIS-RTOS v2), tasks: ...>
- Special features: <EEPROM emulation, external watchdog, ...>

## Requirements

STM32CubeCLT <ENTER VERSION — mandatory!>, STM32CubeMX,
VS Code with the extensions from .vscode/extensions.json.

## Build

```bash
cmake --preset Debug
cmake --build --preset Debug
```

Reference values (with the toolchain above):
- Debug:   flash <...> B, RAM <...> B
- Release: flash <...> B

## Flash and debug

```bash
STM32_Programmer_CLI -c port=SWD -w build/Debug/Project123.hex -v -rst
```

Debug: F5 in VS Code ("Debug (ST-Link)").

## Workflow

Change the hardware configuration only through Project123.ioc in CubeMX
(Generate Code; own code only in USER CODE blocks).
Logic changes directly in Core/Src/.

## Versions

- v1.0 (<date>): <what is in it> — built with CubeCLT <version>

## Open points

- <known construction sites, open questions, planned validations>
````

Why the CubeCLT version is mandatory: it makes every build exactly reproducible years
later. Why reference build sizes: a build on another machine that gives the same numbers
(apart from a few bytes with another GCC generation) is the quickest proof that the
environment is set up correctly.

### 2.5 Git and GitHub — what we do and why

First the terms, then the commands:

**Git** keeps a record of every committed state of the project in the hidden subfolder
`.git/`. A committed state is called a **commit** — like a labelled photo of the entire
project folder at one point in time. **GitHub** is the server these commits are uploaded
to (**push**). `main` is the name of the main line (**branch**) of the history. A **tag**
is a label on a specific commit ("this is v1.0").

First create an EMPTY repository on github.com (signed in, "New repository") — without
ticking "Add README", otherwise it collides with our local state right away. Then, in the
project folder:

```bash
git init                       # turns the folder into a Git project (creates .git/)
git add .                      # puts all files into the "basket" for the next commit
                               #   (everything .gitignore does not exclude)
git commit -m "Project skeleton: generated by CubeMX (CMake), STM32CubeCLT <version>"
                               # writes the basket as state no. 1 -
                               #   the message (-m) describes WHAT and WHY
git branch -M main             # names the main line "main" (GitHub standard)
git remote add origin https://github.com/<user>/<repo>.git
                               # links the local project with the GitHub repo
                               #   ("origin" is just the usual nickname for it)
git push -u origin main        # uploads all commits to GitHub; -u remembers
                               #   the link, from now on "git push" is enough
```

**Why the freshly generated state should be the FIRST commit, before any own code:**
then the history separates cleanly, forever, between "this is what the generator made"
and "this is what we changed" — and after every later "Generate Code", `git diff` shows
exactly what CubeMX touched.

The three everyday check commands (they cost nothing and change nothing):

```bash
git status          # Which files are changed/new? Where am I?
git diff            # What EXACTLY changed (line by line)?
git log --oneline   # The history: one commit per line
```

Make it a habit: **`git status` and `git diff` before every commit** — commit what you
have seen and understood, not blindly.

And for releases:

```bash
git tag -a v1.0 -m "v1.0 - built with STM32CubeCLT <version>"
git push origin v1.0
```

Why tag: the tag freezes the exact source state of a delivery. "Which code runs on
customer X's devices?" can then be answered in seconds: `git checkout v1.0` restores
exactly that state.

#### Publishing production states: GitHub releases

Binary files (.hex/.bin/.dfu) do NOT belong in the repository — that is why `build/` is
in .gitignore. Still, a shipped state should stay at hand permanently without having to
rebuild it. That is exactly what **releases** are for: they attach files to a tag — that
is, to exactly the source state they were built from. "Which firmware did customer X
get?" is then one click.

Way 1 — command line (recommended; one command creates the tag, the release page and the
attachment):

```bash
cmake --preset Release
cmake --build --preset Release
gh release create v1.0 build/Release/Project123.hex --title "v1.0" --notes "First production version. Built with STM32CubeCLT <version>."
```

For several files (e.g. also .bin or .dfu) just list them one after another. If the tag
v1.0 already exists, it is used; otherwise `gh` creates it on the current commit.

Way 2 — browser: repo page on github.com → **"Releases"** on the right → **"Draft a new
release"** → choose or create a tag (v1.0) → title and description (toolchain version!)
→ drag the .hex into the attachment field → **"Publish release"**.

The result is at `github.com/<user>/<repo>/releases`: per version the description, the
source state (automatically as zip/tar) and your attached binaries for download.

#### Working together: how a colleague joins — and how to work as a team

**Once, for the new colleague:**

1. **Grant access:** for a private repo on github.com → *Settings → Collaborators → Add
   people* → invite the colleague's GitHub name (public repos can be read by anyone, but
   only invited collaborators may write). The colleague accepts the invitation by e-mail.
2. **Set up the machine** following Part 1 of this guide — important: install the same
   CubeCLT version that is in the project's README.
3. **Get the project and test the environment:**

```bash
git clone --recurse-submodules https://github.com/<user>/<repo>.git
cd <repo>
cmake --preset Debug
cmake --build --preset Debug
```

   `--recurse-submodules` never hurts and is mandatory as soon as the project uses
   submodules (see the FreeRTOS side note). The first build is also the environment
   test: the memory table must match the reference values in the README (± a few bytes
   with another GCC generation).

**The daily rhythm for two — three rules:**

1. **`git pull` before starting work** — fetches the others' commits and merges them
   with your own state. Whoever does not pull in the morning works on an outdated state
   and creates the afternoon's conflicts themselves.
2. **Commit and push small and often** — the longer two states drift apart, the bigger
   the collision at the end.
3. **Only push states that build** — `main` must be buildable for everyone at any time.

**When the push is rejected** (`rejected — fetch first`): not an error — the colleague
was just faster. Sequence:

```bash
git pull        # fetch and merge the other commits
```

Usually Git merges both states automatically (changes in different files or places). If
both touched THE SAME lines, a **merge conflict** occurs: Git marks the spot in the file
with `<<<<<<<` (your state), `=======`, `>>>>>>>` (the colleague's state). VS Code offers a
merge editor with buttons ("Accept Current/Incoming Change" or combine both). Procedure:
decide the conflict on content, the markers must disappear completely, then **build** (the
merged state must compile!), `git add <file>`, `git commit`, `git push`.

**Keeping conflicts small — two project-specific agreements:**

- The peripheral-files option (see side note) spreads the code over many files — whoever
  works on different peripherals does not collide in the first place.
- **Only one person changes the `.ioc` at a time.** Merging an ioc by hand is error-prone,
  and a CubeMX Generate changes many files at once. Agreement: announce the hardware change
  → generate → build → push immediately; the other person pulls before continuing.

**When to use branches and pull requests?** For two people with pull discipline, working
directly on `main` works well. When the team grows or a risky change is coming, the next
step is the PR workflow — working on a branch of your own:

```bash
git switch -c feature/uart-protocol     # create a branch and switch to it
# ... work and commit as usual ...
git push -u origin feature/uart-protocol
```

On github.com the button **"Compare & pull request"** then appears: the colleague sees
the complete diff of the change, comments directly on the lines, and only after their
approval is it merged into `main` with a button. Two effects: `main` stays guaranteed
clean, and four eyes have seen every change. One rule applies in every mode: **never
`git push --force` on `main`** — it overwrites history that colleagues already have and
does more damage than any merge conflict.

---

## Part 3: The daily work cycle

```
change code -> save -> build -> flash/test -> git status/diff -> commit -> push
```

- **Own code goes EXCLUSIVELY into the `/* USER CODE BEGIN ... */ ... /* USER CODE END
  ... */` blocks** of the generated files — or into entirely separate .c/.h files that
  are added in the root `CMakeLists.txt` under `target_sources(...)`. Only then does it
  survive the next "Generate Code". Code outside the blocks is DELETED WITHOUT WARNING
  when generating.
- Build: `cmake --build --preset Debug` or F7 in VS Code. Guaranteed fresh full build:
  `cmake --build --preset Debug --clean-first`
- Flash: `STM32_Programmer_CLI -c port=SWD -w build/Debug/<name>.hex -v -rst`
  (`-v` = verify after writing, `-rst` = restart afterwards)
- Debug: F5 in VS Code — flashes, starts the GDB server, stops at `main()`. F10 = line by
  line, F5 = continue.
- Commit: `git add -u` (all changed, already versioned files) plus `git add <file>` for
  new files, then `git commit -m "..."` and `git push`.

**Hardware change needed? The complete sequence step by step:**

First, "where do I type this": all `git` and `cmake` commands run in a terminal in the
project folder — most conveniently the **integrated terminal in VS Code** (Ctrl+\`), which
already opens in the right folder. An external terminal or PowerShell window works just
as well; the commands are identical. For almost everything there is also a click path in
VS Code; both are given below.

**Step 0 — clean starting point:**

```bash
git status        # should say: "nothing to commit"
```

If own changes are still open: commit them first. Only then does the diff afterwards show
EXCLUSIVELY what the generator did.

**Step 1 — change:** open the `.ioc` in CubeMX, adjust the hardware (pin, peripheral,
clock), **GENERATE CODE**.

**Step 2 — look at what the generator did:**

```bash
git status        # which files were touched?
git diff          # what exactly, line by line? (page: space, quit: q)
```

Click path: open the Source Control view (**Ctrl+Shift+G**) — it lists all changed files;
clicking a file opens the side-by-side comparison (old on the left, new on the right).
Usually more pleasant for beginners than the terminal diff. Expectation: changes only in
generated areas (MX_..._Init, ioc, possibly msp/it) — your own USER CODE blocks must be
untouched.

**Step 3 — build (yes, this matters):**

```bash
cmake --build --preset Debug
```

(or F7.) Only a successful build proves that the newly generated state compiles — NEVER
commit an unbuilt state, otherwise a broken intermediate state sits in the history and
the next `git checkout` onto it falls flat.

**Step 4 — commit:**

```bash
git add -u
git commit -m "CubeMX: add UART3"
git push
```

Never mix generator changes and manual changes in the same commit — first the CubeMX
commit, then separately the code that uses the new hardware.

**And if something went wrong? Undo in three levels:**

*Level 1 — the changes are NOT committed yet* (the most common case, e.g. generated the
wrong thing in CubeMX by accident):

```bash
git restore Core/Src/main.c   # reset one file to the last commit
git restore .                 # discard ALL open changes (careful: final!)
```

Click path: Source Control view → arrow icon ("Discard Changes") on the file, or at the
top for all.

*Level 2 — the change is already committed:* do not delete it, reverse it — `git revert`
creates a new commit that exactly undoes the old one. The history stays complete and
honest:

```bash
git log --oneline             # list the history: one commit per line,
                              #   the short hash first (e.g. a1b2c3d); quit: q
git revert a1b2c3d            # reverse this commit (confirm the editor)
git push
```

*Level 3 — just look at how it used to be:*

```bash
git log --oneline                          # find the hash of the state you want
git checkout a1b2c3d                       # look at the complete old state ("detached HEAD" -
                                           #   a read-only trip, do not commit here!)
git checkout main                          # back to the present
git restore --source a1b2c3d Core/Src/main.c   # or: bring just ONE file from an
                                               #   old commit into the present
```

Click path for a file's history: select the file in the Explorer → expand the
**"Timeline"** at the bottom of the side bar — it lists all commits that changed this
file; a click shows the comparison. (Even more convenient with the optional GitLens
extension from extensions.json.)

What we deliberately do NOT need: `git reset --hard` throws committed history away
irretrievably — our workflow has no reason for it; `revert` and `restore` cover
everything and stay traceable.

#### Example: a complete run from Debug to delivery

The most important insight first: **switching from Debug to Release changes NOT A SINGLE
source file.** That is exactly what the presets are for — the same code, two
translations. At most, paths in commands change (`build/Debug/` → `build/Release/`).

**Step 1 — develop, for days or weeks, all in Debug:**

```bash
cmake --preset Debug              # once per project
cmake --build --preset Debug      # after every change (or F7)
```

F5 flashes and debugs the Debug build; `DBG_MSG(...)` output is active. Commit and push
intermediate states as usual.

**Step 2 — build the Release (identical source state!):**

```bash
cmake --preset Release            # once per project
cmake --build --preset Release
```

The result is in `build/Release/`: a much smaller .hex, and all `#ifdef DEBUG` output is
gone automatically — without touching any file.

**Step 3 — flash THIS build and validate it on the hardware:**

```bash
STM32_Programmer_CLI -c port=SWD -w build/Release/Project123.hex -v -rst
```

You validate the Release, not the Debug — because the Release is what ships (the
optimisation produces a different machine program, see section 2.2).

**Step 4 — commit and publish:**

```bash
git tag -a v1.0 -m "v1.0 - built with STM32CubeCLT <version>"
git push origin v1.0
gh release create v1.0 build/Release/Project123.hex --title "v1.0" --notes "First production version."
```

**Where paths change — and where nothing is touched:**

- `launch.json` deliberately points to `build/Debug/…` and stays that way: Release builds
  are hardly debuggable because of the optimisation. Only exception — a fault that ONLY
  happens in Release: for that, enable the commented-out configuration "Debug the Release
  build" in launch.json (and for usable symbols `-g0` → `-g3`, see section 2.4).
- Flash command: path depending on purpose, `build/Debug/` (development) or
  `build/Release/` (validation/delivery).
- Source code, CMakeLists, presets, .ioc: **unchanged** — the entire difference is in the
  choice of preset.

**Production state:** Release build from the terminal (`cmake --preset Release`, then
`cmake --build --preset Release`), validate on hardware, tag the version, update the
README (sizes, toolchain) and publish the .hex as a GitHub release (section 2.5).

---

## Side note: FreeRTOS — via the CubeMX checkbox or by hand?

There are two legitimate ways for RTOS projects; for the standard described here, **way A
is the normal case**, way B needs a concrete reason.

**Way A — checkbox in CubeMX** (Pinout & Configuration → Middleware → FREERTOS, interface
CMSIS_V2): CubeMX puts a copy of the kernel under `Middlewares/`, generates
`FreeRTOSConfig.h` from the ioc settings, automatically switches the HAL time base to a
hardware timer (SysTick then belongs to the RTOS) and generates task skeletons with the
CMSIS-RTOS2 API (`osThreadNew`, `osDelay`). *Pros:* zero integration effort, configuration
by GUI, survives regeneration, a combination tested by ST. *Cons:* the kernel version is
tied to the STM32Cube firmware package (usually not the newest), and the CMSIS-RTOS2
layer sits as a wrapper between code and kernel — convenient, but an indirection that
does not pass through every FreeRTOS feature.

**Way B — integrate the kernel yourself** (do NOT tick the box) is worth it if at least
one of these reasons applies: a current kernel version is needed (bug fixes, new
features); the native FreeRTOS API is wanted (`xTaskCreate` instead of the wrapper); the
kernel version should be versioned by the project itself (Git submodule with a version tag
= reproducible and identical across several projects); or features beyond the wrapper are
used (static allocation throughout, stream buffers directly, kernel hooks).

Way B in short:

1. **CubeMX:** do not enable FREERTOS, but still set *SYS → Timebase Source = TIM1* (or
   similar) so that SysTick stays free for the RTOS. In the NVIC code generation settings,
   untick generation of the SVC, PendSV and SysTick handlers — these three vectors belong
   to the kernel.
2. **Get the kernel as a versioned submodule** (the version tag makes the state
   reproducible):

```bash
git submodule add https://github.com/FreeRTOS/FreeRTOS-Kernel.git Middlewares/FreeRTOS-Kernel
cd Middlewares/FreeRTOS-Kernel
git checkout V11.1.0
cd ../..
```

   Note for colleagues: a repo with submodules is cloned with
   `git clone --recurse-submodules …` (or afterwards `git submodule update --init`).
3. **Create `Core/Inc/FreeRTOSConfig.h` yourself** (templates are in the kernel repo under
   examples). Three lines in it save any handler fiddling, because they wire the kernel
   routines directly to the Cortex-M vectors:

```c
#define vPortSVCHandler     SVC_Handler
#define xPortPendSVHandler  PendSV_Handler
#define xPortSysTickHandler SysTick_Handler
```

4. **Root CMakeLists** (in the user area, survives regeneration) — the kernel comes with
   official CMake support:

```cmake
set(FREERTOS_PORT GCC_ARM_CM0 CACHE STRING "")   # choose the port for the core (M0+: CM0, M4F: CM4F, ...)
add_library(freertos_config INTERFACE)
target_include_directories(freertos_config INTERFACE Core/Inc)
add_subdirectory(Middlewares/FreeRTOS-Kernel)
target_link_libraries(${CMAKE_PROJECT_NAME} freertos_kernel)
```

5. **Create tasks with the native API** (`xTaskCreate`, `vTaskDelay`, …) — as always in
   USER CODE blocks or separate files.

What changes compared with way A: no `Middlewares/Third_Party/` folder from CubeMX, no
CMSIS-RTOS2 files, `FreeRTOSConfig.h` is maintained by hand instead of generated from the
ioc — in return the project decides the kernel version itself. Do not mix: either CubeMX
manages the RTOS or the project does; both at once lead to duplicate kernel copies and
handler conflicts.

**Why a submodule instead of "download and drop it in"?** A submodule does not store the
foreign files in your repo, only a POINTER to an exact commit/tag of the foreign repo —
the version is then technically guaranteed, not merely noted somewhere. Updates are a
deliberate mini commit ("V11.1.0 → V11.2.0") instead of a giant diff from
delete-folder-and-copy-again, in which local patches would be silently overwritten; and
several projects can provably point to exactly the same kernel state. Copying
("vendoring") is right where a GENERATOR maintains the foreign code — that is exactly
why the HAL lives in the repo under `Drivers/` as a copy managed by CubeMX (version in the
ioc, renewed only via Generate, the repo stays buildable offline). Rule of thumb:
generator-maintained → copy in the repo; self-managed foreign code → submodule with a
version tag.

**What an update looks like — both ways compared:**

*Update with way A (CubeMX manages kernel and HAL):* the version is tied to the STM32Cube
firmware package. Sequence:

1. Clean state: `git status` must be empty — otherwise commit first.
2. CubeMX: *Help → Manage Embedded Software Packages* → install a newer version of the
   package (e.g. STM32Cube FW_G0). Then open the `.ioc` — CubeMX offers to migrate to the
   new package — confirm and **Generate Code**.
3. Now the diff work. The diff is huge (thousands of lines in `Drivers/` and
   `Middlewares/`) — nobody reads it line by line, and nobody needs to. Instead, three
   looks:

```bash
git diff --stat                              # look 1: overview - WHICH files,
                                             #   how many lines? (expected: almost
                                             #   everything in Drivers/ and Middlewares/)
git diff -- ':!Drivers' ':!Middlewares'      # look 2: the diff WITHOUT the generated
                                             #   libraries - what remains is only what
                                             #   touches your own area
git diff -- Core/                            # look 3: specifically your own files -
                                             #   the USER CODE blocks must be
                                             #   unchanged
```

   The exclamation-mark notation `':!folder'` hides paths from the diff — the most
   important diff trick for generator updates.
4. Build, test against the hardware, then as ONE commit:
   `git commit -m "CubeMX: FW package G0 1.6.x -> 1.7.x"`. If something shows up later:
   `git revert` of this one commit cleanly undoes the whole update.

*Update with way B (submodule):* here the update is a deliberate change of pointer — and
the diff in the main project consists of ONE line:

```bash
cd Middlewares/FreeRTOS-Kernel
git fetch --tags                             # get new version tags
git tag --list "V11.*"                       # what is there?
git log --oneline V11.1.0..V11.2.0           # what changed in the kernel?
                                             #   (plus read the CHANGELOG/release notes)
git checkout V11.2.0
cd ../..
git diff                                     # shows exactly one change:
                                             #   -Subproject commit abc123...
                                             #   +Subproject commit def456...
git add Middlewares/FreeRTOS-Kernel
git commit -m "FreeRTOS-Kernel V11.1.0 -> V11.2.0"
git push
```

   Afterwards, again: build and test against the hardware. If you do not like the result,
   the way back is trivial: `git checkout V11.1.0` in the submodule again and commit the
   pointer — or simply `git revert` the update commit.

The difference in one sentence: with way A you trust the generator and check with an
exclusion diff that your own area stayed untouched; with way B the update itself is only
one line, and what changed in content you read specifically in the foreign repo's log.

---

## Side note: peripheral init as separate .c/.h files per peripheral

In the CubeMX Project Manager, tab **Code Generator**, there is the checkbox **"Generate
peripheral initialization as a pair of '.c/.h' files per peripheral"**. It decides WHERE
the generated initialisation code ends up — and with that it sets the course for the
readability of the whole project.

**Way A — box UNticked (CubeMX default):** all `MX_..._Init()` functions, all peripheral
handles (`hadc1`, `huart2`, …) and their USER CODE blocks end up together in `main.c`.
*Pro:* everything in one place, one file less to think about — fine for mini projects
with two peripherals. *Con:* `main.c` quickly grows to many hundreds of lines, application
logic and hardware initialisation mix, and after every "Generate Code" the `git diff` of
main.c is big and confusing, because generator and hand-written code live in the same
file.

**Way B — box ticked (our standard for new projects):** CubeMX creates a file pair per
peripheral in `Core/`: `gpio.c/.h`, `adc.c/.h`, `usart.c/.h`, `tim.c/.h` and so on. Each
holds the `MX_..._Init()` function, the handle (available as `extern` in the header) —
and **its own USER CODE blocks per file**. `main.c` shrinks to clock setup and the init
calls.

*What changes, or has to be adjusted:*

- **Nothing in the build:** CubeMX adds the new files to
  `cmake/stm32cubemx/CMakeLists.txt` itself — building, flashing and debugging work
  unchanged.
- **Your own code gets a natural home:** helper functions for a peripheral belong in ITS
  file (e.g. a `UART_SendDebugLine()` in the USER block of `usart.c`, not in main.c). That
  keeps main.c about the application logic.
- **`git diff` after "Generate Code" becomes precise:** an ADC change in CubeMX changes
  `adc.c/.h` — and hardly anything else. Generator commits become small and
  self-explanatory.

*Pros* overall: structure and findability ("where is the UART?" → `usart.c`), small and
readable diffs, fewer conflicts in team work, peripheral code can be reused on its own.
*Cons:* more files (cosmetic), and the handles are no longer in main.c — whoever needs
`hadc1` in their own file includes `adc.h` (instead of an `extern` by hand).

**Converting an EXISTING project** (ticking the box later) works but needs care: commit
first (clean state!), tick the box, Generate Code, then check with `git diff` — the init
functions move from main.c into the new files, and **own code that was in main.c inside
the moved init functions (e.g. in the USER block of MX_ADC1_Init) has to be moved by hand
into the USER blocks of the new file**. Then build and commit as a separate commit
("CubeMX: peripheral init in separate files"). For running, stable projects the
conversion is usually not worth it — tick the box from the start in every NEW project.

---

## Part 4: Checklist for every new project

- [ ] CubeMX: toolchain = CMake, name AND path without spaces/umlauts
- [ ] Code Generator: box "pair of '.c/.h' files per peripheral" ticked
- [ ] First build runs (terminal), memory table noted in the README
- [ ] .hex/.bin block appended to the end of the root CMakeLists (section 2.3)
- [ ] .gitignore, .vscode/, README.md added (including the CubeCLT version)
- [ ] On Windows: `git config --global core.autocrlf input` set (once per machine)
- [ ] Generated state = first commit, repo on GitHub, push successful
- [ ] Own code only in USER CODE blocks
- [ ] No empty counting loops for timing — GCC optimises them away completely! Always use
      HAL_Delay or a hardware timer.
- [ ] Timing constants as named #defines, not bare numbers
- [ ] With flash EEPROM emulation: reserve the pages in the linker script (shorten the
      FLASH LENGTH)
- [ ] Release build tested, validated on hardware, version tagged and published as a
      GitHub release (.hex attached)

---

## Part 5: Troubleshooting — typical errors and how to fix them

**F5: "ST-LINK: GDB Server Quit Unexpectedly"**
Two common causes. (1) No ST-Link connected — then the message is normal; details are in
the terminal tab "gdb-server". (2) Cortex-Debug passes a wrong CubeProgrammer path to the
GDB server — visible in the debug log as `-cp` with a path that does not exist on the
machine (the default of a separate CubeProgrammer installation instead of CubeCLT). Fix in
the project's `.vscode/settings.json`:

```jsonc
{
    // Linux:
    "cortex-debug.stm32cubeprogrammer": "/opt/st/stm32cubeclt_<version>/STM32CubeProgrammer/bin"
    // Windows:
    // "cortex-debug.stm32cubeprogrammer": "C:\\ST\\STM32CubeCLT_<version>\\STM32CubeProgrammer\\bin"
}
```

Then reload the window; in the next debug log `-cp` must point to the CubeCLT path. This
settings.json is also the right place for the IntelliSense setting (see the
cpptools/clangd entry) — both versioned together in the project.

**"cannot execute binary file: Exec format error" or "No compiler found in cache file"**
The wrong play button was pressed: the play/bug icons at the bottom in the STATUS BAR
belong to CMake Tools and try to run or debug the program ON THE PC — but the .elf is ARM
machine code for the controller, that cannot work (both messages are harmless, nothing is
damaged). The right way to flash/debug: open the Run & Debug VIEW (Ctrl+Shift+D), choose
the configuration in the dropdown at the top, then F5. Rule: status bar at the bottom =
build (F7); Run & Debug view on the left = onto the controller (F5).

**Warning: "cpptools extension and stm32-cube-clangd extension enabled ... conflict"**
The ST extension installs its own IntelliSense engine (stm32-cube-clangd); together with
the one of the C/C++ extension, two engines then provide completion and error display at
the same time. Fix: click the button offered in the message to disable Microsoft
IntelliSense — or manually set "C_Cpp: Intelli Sense Engine" to `disabled` in the settings
(tab "Workspace"). clangd is the better choice here: it reads the compile_commands.json
created by the CMake build and so knows exactly the flags and defines of the real build.
Building and debugging are not affected by the choice. Self-test that clangd works
cleanly: type `HAL_GPIO_` in main.c as a test — completion must offer the HAL functions;
a deliberate typo gets a red squiggle, and the mark disappears as soon as it is fixed.
Then the editor display is coupled to the real build.

**The extension recommendation prompt does not appear**
Three possible causes: (1) all recommended extensions are already installed — then there
is nothing to ask (extensions apply to all of VS Code, not per project). (2) extensions.json
is in the wrong place (see the double .vscode below). (3) The pop-up was dismissed earlier.
The reliable way without the pop-up: Extensions view (Ctrl+Shift+X) → type `@recommended`
into the search field → the section "Workspace Recommendations" shows the entries from the
file, the cloud icon next to it installs all missing ones in one go. If the section is
empty, the file is not being read: check its location or syntax (missing/extra comma).

**launch.json: "Property servertype/device/... is not allowed" and "The debug type is not recognized"**
That almost never means "wrong field" but almost always: the matching debug extension is
missing or not loaded — without Cortex-Debug, VS Code does not know the type
`cortex-debug` and checks the file against the wrong schemas. Install/enable Cortex-Debug,
reload the window ("Developer: Reload Window") — the warnings disappear.

**Files in .vscode/ are ignored (configurations show up nowhere)**
Classic: when creating them in the Explorer panel, the .vscode folder was selected, and a
NESTED folder `.vscode/.vscode/` was created. Check and fix (Linux; on Windows move the
files in the Explorer):

```bash
ls .vscode                      # should show: extensions.json  launch.json
mv .vscode/.vscode/* .vscode/ && rmdir .vscode/.vscode   # if nested
```

**Linux: `bash: code: command not found`**
VS Code is installed as a Flatpak (check: `flatpak list | grep -i code`). Remove the
Flatpak and install the RPM version from step 1.3 — the Flatpak sandbox does not get along
with the toolchain and the ST-Link.

**Linux: `arm-none-eabi-gcc: command not found` right after installing CubeCLT**
The PATH only takes effect after logging out and back in. If in doubt, check that the files
are there: `ls /opt/st/`.

**Windows: `arm-none-eabi-gcc` / `cmake` / `code` / `git` "is not recognized"**
Almost always: the PowerShell window was already open before the tool was installed. PATH
changes only apply to NEWLY opened windows — close the window, open a new one. The same
goes for the terminal INSIDE VS Code: restart VS Code after installations.

**Linux: `sudo` says "Sorry, try again"**
That is just a wrong password for sudo — not an error of the actual command. Many commands
(e.g. `rpm --import`) print NOTHING at all on success: no output = good.

**`unzip: cannot find or open ...` / file not found**
The file is not where the command looks for it. Find it: on Linux
`ls ~/Downloads/ | grep -i <part-of-name>` or
`find ~ -name "*<part-of-name>*" 2>/dev/null`; on Windows `dir $env:USERPROFILE\Downloads`
or look in the Explorer where the browser saved it. Then adjust the path in the command.
(In PowerShell the Tab key completes file and folder names.)

**`ninja: no work to do` — although files were changed/replaced**
Ninja decides by timestamp; unpacked or copied files often carry an old date. Fix:
`cmake --build --preset Debug --clean-first` (on Linux alternatively
`touch Core/Src/*.c Core/Inc/*.h` and rebuild). Rule: after unpacking anything over a
project, do one clean rebuild.

**The build uses the wrong compiler (x86 errors, a huge .elf, linker errors about system libraries)**
A "kit" was selected in VS Code (e.g. /usr/bin/gcc, or a Visual Studio or MinGW compiler
on Windows). Fix: Ctrl+Shift+P → "CMake: Select a Kit" → **[Unspecified]** → "CMake:
Delete Cache and Reconfigure". Check: the Output must say
`Found assembler: /opt/st/stm32cubeclt_.../arm-none-eabi-gcc` (Windows:
`C:/ST/STM32CubeCLT_.../arm-none-eabi-gcc`).

**ST extension: "Checking toolchain step failed" / bundle errors**
A known hiccup of the extension; it does NOT affect building — the terminal and CMake Tools
work independently of it. Reload the window ("Developer: Reload Window"); if needed,
uninstall and reinstall the extension. It often resolves itself after the first successful
terminal build.

**`STM32_Programmer_CLI -l` finds no ST-Link**
Unplug and replug the USB (on Linux, udev rules only take effect on re-enumeration). Other
candidates: a charge-only cable instead of a data cable, a different USB port, the target
board without power (the ST-Link usually does not power the target). On Linux: udev rules
declined during installation (run the CubeCLT installer again). On Windows: an "ST-Link"
must appear under USB devices in the Device Manager — if it is missing, run the CubeCLT
installer again (driver option).

**Windows: antivirus/SmartScreen blocks the installer or arm-none-eabi-gcc**
ST installers are signed — with SmartScreen choose "More info → Run anyway". With company
virus scanners, have C:\ST\ and the project folder added as exceptions (build tools that
create many files quickly are often wrongly flagged and slow the build down massively).

**`git commit` → "Please tell me who you are"**
Set `git config --global user.name/user.email` once (step 1.4).

**`git push` → authentication error / password rejected**
GitHub does not accept account passwords. Run `gh auth login` (step 1.4); after that pushing
works permanently.

**`git status`: "nothing to commit" — although something was changed**
Almost always: the wrong folder, or the change never arrived (e.g. unzipped into another
directory). Check `pwd`, then `git diff` — if it is empty, nothing was actually changed.

**Windows: `git diff` shows ALL lines as changed (line-ending noise)**
`core.autocrlf input` was missing when cloning/creating. Set it (step 1.4); for a repo that
is already affected, clone it fresh from GitHub if in doubt.

**Own code disappeared after "Generate Code"**
It was outside the USER CODE blocks. Restore it with Git: `git diff` shows the loss,
`git checkout -- <file>` brings back the last committed state (hence: commit BEFORE every
Generate!). Then move the code into a USER block.

**Linker error `region 'FLASH' overflowed by N bytes`**
The program is bigger than the flash (possibly shortened for EEPROM emulation). First check
with a Release build (-Os); then shrink the code or reconsider the reservation.

**Linker warnings `_close/_read/_write is not implemented`**
Harmless: C library stubs for file functions that do not exist on the controller. Only
relevant if printf/scanf are to be used — then provide a real output (e.g. UART).

**`HAL_Delay()` hangs or timings are wrong**
Check the time base: is the tick timer/SysTick running? With exotic clocks (very low
frequencies) check the generated time base calculation — the standard formulas assume
≥ 1 MHz (file stm32g0xx_hal_timebase_tim.c).

---

## History

Until October 2026 this guide existed as two German repositories,
`stm32-workflow-linux` and `stm32-workflow-windows`. They were merged into this one and
translated to English.

## License

[MIT](LICENSE).
