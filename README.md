# PAIcom HotSwap Patcher

Injects a file-watching hot-swap runtime into **your own copy** of PAIcom.exe
so you can trigger in-game commands by writing a phrase to a text file — no
microphone required.

> **Legal notice**: This tool contains no game code.  It reads your locally-
> installed copy of PAIcom.exe, patches it in memory, and writes a new file.
> You must own a legitimate copy of PAIcom (e.g. via Steam) to use this tool.

---

## Requirements

| Requirement | Notes |
|-------------|-------|
| Windows x64 | The patcher is self-contained — no .NET install needed |
| .NET Framework 4.8 | Required by PAIcom itself; already present if you can launch the game |
| PAIcom.exe | Your own Steam copy — **not included** |

---

## Quick start

1. Copy `PAIcomPatcher.exe` into the same folder as `PAIcom.exe`
   (usually `SteamLibrary\steamapps\common\PAIcom`).

2. Open a terminal in that folder and run:

   ```
   PAIcomPatcher.exe PAIcom.exe --out PAIcom.patched.exe --backup
   ```

3. Launch `PAIcom.patched.exe` instead of the original (You are able to rename it back to `PAIcom.exe` and remove/rename the backup if you like to something like `PAIcom.original.exe`, but keeping them is recommended for easy troubleshooting).

4. A file called `command_input.txt` is created next to the exe on first run.
   Write any recognised phrase to it (one line, then save) to dispatch the
   matching in-game command:

   ```
   hey paicom open youtube
   ```

5. Watch `hotswap.log` in the same directory for diagnostic output.

---

## Command mapping

Place a file at `custom-commands\commands.txt` (relative to the exe) with one
command per line in this format:

```
hey paicom open youtube (youtube.txt)
hey paicom open google (google.txt)
hey paicom show animations (animations.txt)
```

The part in parentheses is the name of the matching animation script inside
the `animations\` folder (`.txt` extension is stripped automatically).
Matching is case-insensitive.

---

## Animation scripts

Each file in `animations\` can use these directives (one per line):

```
HIDE_ALL
SHOW 3
HIDE 1
OPEN_URL https://example.com
PLAY_AUDIO audio/chime.wav
WAIT 500
```

| Directive | Description |
|-----------|-------------|
| `HIDE_ALL` | Hides every PictureBox on the form |
| `SHOW N` | Shows PictureBox at index N |
| `HIDE N` | Hides PictureBox at index N |
| `OPEN_URL <url>` | Opens the URL in the default browser |
| `PLAY_AUDIO <path>` | Plays a `.wav` file (relative to the exe folder) |
| `WAIT <ms>` | Pauses script execution for N milliseconds |

`SHOW`/`HIDE`/`HIDE_ALL` directly manipulate the game's own PictureBox controls
discovered via reflection at startup.

> Animation scripts are used when the speech engine dispatch chain fails (see
> below).  When `[EMULATE-ASYNC] OK` or `[EMULATE-STOP] OK` appears in the
> log, the game's own full handler ran and played its built-in animations.

---

## Dry run / diagnostics

```
PAIcomPatcher.exe PAIcom.exe --dry-run --verbose
```

Prints what *would* be patched without writing any files.

```
PAIcomPatcher.exe PAIcom.exe --analyze
```

Dumps the top-ranked methods by instruction count, branch density, and call
count — useful for debugging ConfuserEx-obfuscated assemblies.

---

## Building from source

Requires the [.NET 8 SDK](https://dotnet.microsoft.com/download).

```
git clone https://github.com/WentersTers/PAIcom-Patch-Injector.git
cd PAIcom-Patch-Injector
dotnet build PAIcomPatcher.csproj -c Release
```

### How to find the PAIcom folder path

1. Open Steam and go to your Library.
2. Click **PAIcom**.
3. Click the **gear icon** next to the Play button.
4. Choose **Manage → Browse local files**.
5. Copy the folder path from Explorer's address bar.
6. Use that path in the `cd` command below.

### Compiling a self-contained release exe (PowerShell)

```powershell
cd 'C:\path\to\PAIcom'   # paste your PAIcom folder path here

dotnet publish PAIcomPatcher.csproj -c Release -r win-x64 `
    --self-contained true -p:PublishSingleFile=true `
    -p:PublishTrimmed=false -o publish\
```

This bundles .NET 8 into the output so end-users do not need a separate
runtime installed.  The finished `PAIcomPatcher.exe` is written to
the `publish\` sub-folder.

Then:

1. Copy `publish\PAIcomPatcher.exe` into the same folder as `PAIcom.exe`.
2. Run it as described in **Quick start**.

---

## How it works (technical summary)

### 1 — Patch-time: structural IL analysis

Because PAIcom uses ConfuserEx (all strings encrypted, identifiers scrambled),
the patcher cannot search for named types.  Instead it scores every method by
branch density and call count to locate:

- The **`SpeechRecognizedEventArgs` handler** — the game's main command
  dispatcher.
- The **main constructor** (`.ctor`) — used as the injection entry point.

### 2 — Patch-time: Roslyn in-memory compilation

A C# `HotSwapRuntime` class (`Core/HotSwapTemplate.cs`) is compiled against
.NET Framework 4.8 reference assemblies using Roslyn and injected as a new
type directly into the target module.  No external DLLs are dropped on disk.

### 3 — Patch-time: IL injection

Two splice points are written into the patched assembly:

| Splice | What is injected |
|--------|-----------------|
| Main constructor | `HotSwapRuntime.StartWatcher()` call after `base..ctor()` |
| Speech handler | `HotSwapRuntime.OnCommandDispatched(command)` before every `ret` |

### 4 — Runtime: start-up

When `PAIcom.patched.exe` starts, `StartWatcher()` runs and:

- Creates `command_input.txt` if it does not exist.
- Starts a `FileSystemWatcher` on `command_input.txt`.
- Starts a 500 ms polling loop as a write-lock fallback.
- Spawns a background thread that waits 3 seconds then begins searching for
  the `SpeechRecognitionEngine` instance.

### 5 — Runtime: engine discovery

The engine search thread walks every open `Form` via
`Application.OpenForms`, reading instance fields via reflection to find
a `SpeechRecognitionEngine` or `SpeechRecognizer`.  On success it logs:

```
[ENGINE] Ready: System.Speech.Recognition.SpeechRecognitionEngine.EmulateRecognize(string)
[ENGINE] Ready: System.Speech.Recognition.SpeechRecognitionEngine.EmulateRecognizeAsync(string)
[ENGINE] Ready: RecognizeAsyncCancel()
[ENGINE] Ready: RecognizeAsync(System.Speech.Recognition.RecognizeMode)
```

At the same time the form is scanned for:

- **PictureBox controls** (for `SHOW`/`HIDE` in script fallback).
- **Animation method** — any instance method with signature `Task Method(string)`,
  matched by exact return-type name, partial name, or `AsyncStateMachineAttribute`.
- **Command handler method** — heuristic: `void Method(string)` with a
  non-trivial IL body, used as a last-resort direct call.

### 6 — Runtime: command dispatch (5-attempt chain)

When a phrase is written to `command_input.txt` the injected runtime:

1. Reads and trims the file content.
2. Looks up the phrase in `custom-commands/commands.txt` to resolve a command name.
3. Runs the following dispatch chain in order, stopping at the first success:

| Attempt | Strategy | Log prefix |
|---------|----------|------------|
| 1 | **`EmulateRecognizeAsync(phrase)` on the UI thread** — marshalled via `Form.Invoke()`.  Fires the game's own full handler with animations, audio, and URLs. | `[EMULATE-ASYNC]` |
| 2 | **Stop/restart** — calls `RecognizeAsyncCancel()`, then `EmulateRecognize(phrase)`, then `RecognizeAsync(Multiple)`, all on the UI thread.  Used when the engine rejects async emulation. | `[EMULATE-STOP]` |
| 3 | **Direct animation method** — calls the discovered `Task Method(string)` directly on the UI thread via `BeginInvoke`. | `[DIRECT-INVOKE]` |
| 4 | **Command handler method** — calls the discovered `void Method(string)` directly on the UI thread via `BeginInvoke`. | `[CMD-HANDLER]` |
| 5 | **Script fallback** — reads and executes the matching `animations/*.txt` file locally, including real `SHOW`/`HIDE` via reflection on the form's PictureBox controls. | `[SCRIPT]` |

> **Important**: `SpeechRecognitionEngine` is thread-affine — all calls to it
> must happen on the UI thread that created it.  All engine calls are therefore
> marshalled through `Form.Invoke()` rather than called directly from the
> background watcher thread.

---

## Reading the log

`hotswap.log` (written next to the exe) records every step.  A successful
dispatch looks like:

```
[21:34:02.596] [DISPATCH] 'aliexpress' (phrase: 'hey paicom open aliexpress')
[21:34:02.596] [DISPATCH] Thread: HotSwapPoller (ID 6, IsBackground=True)
[21:34:02.596] [DISPATCH] State: engine=True emulate=True emulateAsync=True animMethod=False mainForm=True
[21:34:02.599] [EMULATE-ASYNC] Marshalling EmulateRecognizeAsync("hey paicom open aliexpress") to UI thread …
[21:34:02.601] [EMULATE-ASYNC] OK – game handler will fire asynchronously.
```

If all engine attempts fail the script fallback runs:

```
[21:34:03.015] [SCRIPT] Running E:\...\animations\aliexpress.txt
[21:34:03.022] [OPEN_URL] https://www.aliexpress.com/
[21:34:03.045] [PLAY_AUDIO] E:\...\audio\aliexpress.wav
```

---

## License

MIT — see [LICENSE](LICENSE)

> This tool does not circumvent copy protection, licensing, or DRM.
> ConfuserEx obfuscation is bypassed solely for interoperability purposes.
> Users are responsible for complying with PAIcom's EULA.



