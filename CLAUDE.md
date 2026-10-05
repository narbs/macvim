@AGENTS.md

# CLAUDE.md — MacVim fork

`AGENTS.md` (imported above) is upstream Vim's guidance and applies to
everything under `src/` and `runtime/` that is shared with Vim. This file adds
what is specific to **MacVim**, the Cocoa GUI for Vim. This repo is a personal
fork (`origin` = github.com/narbs/macvim) of github.com/macvim-dev/macvim,
which itself regularly merges `vim/master` (see the "Merge remote-tracking
branch 'vim/master'" commits). Keep MacVim-only changes in MacVim files where
possible, so that upstream Vim merges stay clean.

## Platform note

MacVim builds and runs **only on macOS with Xcode**. On a Linux checkout you
can read and edit the code but cannot compile or run the GUI. Never claim a
MacVim change was built or tested unless it was done on a Mac.

## Build and test (macOS)

    cd src
    ./configure --enable-gui=macvim [--disable-sparkle]   # see src/MacVim/README
    make                       # builds Vim, then `xcodebuild` -> MacVim.app
    make macvim-tests          # XCTest suite (src/MacVim/MacVimTests)
    make test                  # Vim's script tests, run against MacVim's Vim binary

The app bundle ends up in `src/MacVim/build/Release/MacVim.app`, with
`Contents/MacOS/MacVim` (the GUI) and `Contents/MacOS/Vim` (the editor).
CI is in `.github/workflows/ci-macvim.yaml` and `macvim-buildtest.yaml`.
If you add or remove files in the Xcode project, regenerate the compat
project with `make -C src macvim-xcodeproj-compat` (CI checks this).

## Architecture (read `src/MacVim/README` for the full version)

There are two executables. They talk over **Distributed Objects (DO)**, and
the interface is defined in `src/MacVim/MacVim.h`:

- **MacVim (the frontend, one process)** draws everything, manages windows,
  and handles input. It has no knowledge of editor state.
  - `MMAppController`: NSApp delegate and singleton. It owns the app
    lifecycle (`applicationShouldTerminate:` etc.), launches Vim processes
    (`launchVimProcessWithArguments:workingDirectory:`), handles
    `connectBackend:pid:`, and runs the preload cache of Vim processes
    (`cachedVimControllers`, `preloadPid`). Startup arguments for each launching
    Vim are queued in `pidArguments` (keyed by PID) and are delivered when that
    Vim checks in.
  - `MMVimController`: frontend proxy for one Vim process. It provides
    `sendMessage:data:` (async), `evaluateVimExpression:` (synchronous DO
    with a timeout, like `--remote-expr`), and `objectForVimStateKey:`
    (state that Vim pushes over).
  - `MMWindowController` / `MMWindow` / `MMFullScreenWindow`: the
    NSWindow for one Vim process, covering sizing, full screen, and autosaved
    position (`MMTopLeftPointKey`).
  - `MMVimView`, `MMTabline/`, `MMCoreTextView`, `MMTextViewHelper`: the
    tabline, text rendering, and input.
- **Vim (the backend, one process per window)** is the normal Vim built with
  `FEAT_GUI_MACVIM`.
  - `gui_macvim.m`: the `gui_mch_*` hooks. These queue messages to the
    frontend.
  - `MMBackend`: the DO endpoint. It handles messages such as
    `TerminateNowMsgID` (which calls `getout(0)`) and `evaluateExpression:`,
    and pushes state in `queueVimStateMessage`.
- `MacVim.h`/`MacVim.m`: code shared by both sides, including the message ID
  list (`MSG(...)` X-macro).

A single MacVim window corresponds to exactly one Vim process, which may have
several Vim tabs and splits. All editor state lives in that Vim process.

## MacVim code conventions

- The app target uses **manual retain/release (MRC)**: `retain`, `release`,
  `autorelease`, and `[super dealloc]`. Only the test target uses ARC.
  Follow the existing memory-management style in the file.
- Objective-C style: 4-space indent. Method bodies open `{` on their own line,
  and control flow (`if`/`while`) keeps `{` on the same line. This differs
  from Vim's C files. Match the surrounding file, and keep logging through
  `ASLogDebug/Info/Notice`.
- User defaults keys are declared in `Miscellaneous.h`, defined in
  `Miscellaneous.m`, and registered with their defaults in
  `+[MMAppController registerDefaults]`. Settings UI lives in
  `MMPreferenceController` and `Base.lproj/Preferences.xib`.
- Guard newer macOS APIs with `@available(macos N, *)` and, where needed,
  `#if MAC_OS_X_VERSION_MAX_ALLOWED >= ...`. MacVim supports old macOS
  versions.
- DO re-entrancy is a real hazard: synchronous DO calls and modal loops spin
  the run loop. See "Distributed Object dangers" in `src/MacVim/README`.
- Treat changes to `MainMenu.xib` and the localized `*.lproj` strings with
  care. Menus are translated in `runtime/lang/macvim_menu/` as well.
- User-facing MacVim docs are in `runtime/doc/gui_mac.txt`.
