# PyCharm Recovery Content Pack

## Project One-Liner

Recovered a broken Windows PyCharm setup by separating stale startup lock state from settings migration residue, then restoring Conda interpreter definitions and the CodeGlancePro minimap from JetBrains backup files.

## Resume STAR

### Situation

PyCharm failed to start on Windows with `DirectoryLock$CannotActivateException`, reporting that this process was still running:

```text
C:\Program Files (x86)\JetBrains\PyCharm Community Edition 2022.3.2\bin\pycharm64.exe
```

The reported PID was `48464`, but `Get-Process` and Task Manager did not show a visible PyCharm process. After partial recovery, the IDE lost all Python interpreters and later lost the CodeGlancePro editor minimap.

### Task

Recover the IDE without reinstalling PyCharm or recreating the Python environment, while preserving the intended Conda/Python 3.11 project setup and restoring the missing minimap.

### Action

- Verified the reported process theory with PowerShell:

```powershell
Get-Process -Name pycharm64 -ErrorAction SilentlyContinue |
  Select-Object Id, ProcessName, Path, StartTime, Responding
```

- Identified `%APPDATA%\JetBrains\PyCharm2026.1\migrate.config` as a stale reset/migration marker and removed it:

```powershell
Remove-Item -LiteralPath `
  "$env:APPDATA\JetBrains\PyCharm2026.1\migrate.config" `
  -Force
```

- Distinguished the missing `%APPDATA%\JetBrains\PyCharm2026.1\.lock` from the real startup blocker, `%LOCALAPPDATA%\JetBrains\PyCharm2026.1\.port`, after this error:

```text
java.nio.file.FileSystemException:
%LOCALAPPDATA%\JetBrains\PyCharm2026.1\.port:
The system cannot access the file.
```

- Removed the stale `.port` file:

```powershell
Remove-Item -LiteralPath `
  "$env:LOCALAPPDATA\JetBrains\PyCharm2026.1\.port" `
  -Force
```

- Restored interpreter definitions from:

```text
%APPDATA%\JetBrains\PyCharm2026.1-backup\2026-06-23-14-08\options\jdk.table.xml
```

to:

```text
%APPDATA%\JetBrains\PyCharm2026.1\options\jdk.table.xml
```

using:

```powershell
Copy-Item `
  -LiteralPath "$env:APPDATA\JetBrains\PyCharm2026.1-backup\2026-06-23-14-08\options\jdk.table.xml" `
  -Destination "$env:APPDATA\JetBrains\PyCharm2026.1\options\jdk.table.xml" `
  -Force
```

- Corrected the project interpreter in `<project-root>\.idea\misc.xml` from:

```xml
project-jdk-name="uv (example-project)"
```

back to:

```xml
project-jdk-name="Python 3.11"
```

- Verified the restored interpreter target:

```text
<conda-install>\python.exe
Python 3.11.13
```

- Removed the temporary `uv (example-project)` interpreter only after verifying the Conda interpreter.
- Restored CodeGlancePro from the JetBrains backup:

```text
...\PyCharm2026.1-backup\2026-06-23-14-08\plugins\CodeGlancePro
...\PyCharm2026.1-backup\2026-06-23-14-08\options\CodeGlancePro.xml
```

with:

```powershell
Copy-Item `
  -LiteralPath "...\PyCharm2026.1-backup\2026-06-23-14-08\plugins\CodeGlancePro" `
  -Destination "$env:APPDATA\JetBrains\PyCharm2026.1\plugins\CodeGlancePro" `
  -Recurse `
  -Force
```

and:

```powershell
Copy-Item `
  -LiteralPath "...\PyCharm2026.1-backup\2026-06-23-14-08\options\CodeGlancePro.xml" `
  -Destination "$env:APPDATA\JetBrains\PyCharm2026.1\options\CodeGlancePro.xml" `
  -Force
```

- Checked `disabled_plugins.txt` to make sure CodeGlancePro was not disabled.
- Repeated the `.port` cleanup after the same access error returned following plugin restoration.

### Result

PyCharm opened successfully, the project used the restored Python 3.11 Conda interpreter, the temporary `uv (example-project)` interpreter was removed, CodeGlancePro was restored, the editor minimap was visible again, and the stale `.port` startup lock was cleared.

## Resume Bullets

- Recovered a broken Windows PyCharm installation by tracing `DirectoryLock$CannotActivateException` to stale JetBrains lock state instead of reinstalling the IDE.
- Restored missing Conda/Python 3.11 interpreter definitions by comparing and copying JetBrains `jdk.table.xml` backups into the active PyCharm settings directory.
- Corrected project-level interpreter drift in `.idea\misc.xml`, replacing `uv (example-project)` with `Python 3.11` after global interpreter restoration.
- Restored the CodeGlancePro minimap by recovering the plugin directory and `CodeGlancePro.xml`, then confirming the plugin was not listed in `disabled_plugins.txt`.

## LinkedIn Post

PyCharm recovery can look like one broken IDE, but this case turned into three separate root causes.

First, PyCharm failed with `DirectoryLock$CannotActivateException` and claimed `pycharm64.exe` was still running. PowerShell and Task Manager did not show a live process, so the real problem was not a visible window. The startup blocker was `%LOCALAPPDATA%\JetBrains\PyCharm2026.1\.port`.

Then Reset Settings failed because `%APPDATA%\JetBrains\PyCharm2026.1\migrate.config` already existed. That marker explained a dirty reset state, but it was not the same as the `.port` startup lock.

After clearing `.port`, PyCharm opened, but every interpreter was gone. The Conda environment was still there; PyCharm had lost its interpreter definitions. Restoring `options\jdk.table.xml` from `%APPDATA%\JetBrains\PyCharm2026.1-backup\2026-06-23-14-08` brought the global list back. The project still needed `.idea\misc.xml` changed from `uv (example-project)` back to `Python 3.11`.

Finally, the editor minimap was missing because CodeGlancePro had not been restored. Copying the plugin folder and `CodeGlancePro.xml`, then checking `disabled_plugins.txt`, brought it back.

The useful habit was separating symptoms:

- `migrate.config` was a reset/migration marker.
- `.port` was the startup lock blocker.
- `jdk.table.xml` restored global interpreters.
- `.idea\misc.xml` selected the project interpreter.
- CodeGlancePro controlled the minimap.

That avoided a reinstall and preserved the working Conda setup.

## Short LinkedIn Version

Recovered PyCharm on Windows without reinstalling it.

The key was separating three issues that looked like one failure:

- `migrate.config` was a stale reset/migration marker.
- `%LOCALAPPDATA%\JetBrains\PyCharm2026.1\.port` was the real startup lock blocker.
- Missing interpreters required restoring `jdk.table.xml` and correcting `.idea\misc.xml`.
- The missing minimap was CodeGlancePro, restored from the JetBrains backup and checked against `disabled_plugins.txt`.

Final state: PyCharm opened, Python 3.11 / Conda was restored, `uv (example-project)` was removed, and the CodeGlancePro minimap returned.

## Commit Message

```text
docs: rewrite PyCharm recovery public content

- restructure README as a full public project page
- convert Medium draft into a chronological SEO article
- add reusable resume, LinkedIn, PR, and extension content pack
- preserve all debugging commands, paths, errors, and root-cause distinctions
```

## Pull Request Description

### Summary

This PR rewrites the public content for the PyCharm Windows recovery case study.

### What Changed

- Reworked `README.md` into a structured project document with summary, audience, environment, preserved debug narrative, root-cause map, final recovery sequence, and final state.
- Rewrote `medium-pycharm-debugging-article.md` as a standalone Medium-style story with alternate titles, SEO metadata, tags, and the full chronological debug path.
- Added `docs/content-pack.md` with resume STAR content, resume bullets, LinkedIn posts, commit message, PR copy, and future extension ideas.

### Preserved Technical Details

- `DirectoryLock$CannotActivateException`
- `pycharm64.exe` path and PID `48464`
- `Get-Process` PowerShell check
- `migrate.config` reset marker failure
- `.port` versus `.lock` root-cause distinction
- `Files.deleteIfExists` and `DirectoryLock.lockOrActivate`
- `jdk.table.xml` comparison and restoration
- `uv (example-project)` temporary interpreter
- `.idea\misc.xml` project interpreter correction
- `<conda-install>\python.exe` and `Python 3.11.13`
- CodeGlancePro plugin and options restoration
- `disabled_plugins.txt` check
- repeated `.port` cleanup after plugin restoration

### Validation

- Confirmed README asset references still point to existing files under `assets/`.
- Confirmed the Medium article remains chronological and within the requested 1500-2500 word range.
- Confirmed all preserved commands, error messages, paths, and root-cause distinctions remain present.

## Future Extensions

1. Add a Windows troubleshooting checklist that starts with process verification, then lock-file verification, then settings backup review.
2. Add a PowerShell script that safely reports the presence of `migrate.config`, `.port`, `.lock`, `jdk.table.xml`, `CodeGlancePro.xml`, and `disabled_plugins.txt` without deleting anything.
3. Add a second script mode that performs cleanup only after the user confirms PyCharm is closed.
4. Add screenshots of PyCharm interpreter settings before and after restoring `jdk.table.xml`.
5. Add a small diagram showing the difference between global interpreter definitions and project-level interpreter selection.
6. Add a recovery matrix for other JetBrains IDEs that use similar config, system, plugin, and backup directory layouts.
7. Add a checklist for safely removing temporary interpreters like `uv (example-project)` only after the intended Conda interpreter has been verified.
8. Add a CodeGlancePro-specific recovery note covering plugin directory restoration, `CodeGlancePro.xml`, and `disabled_plugins.txt`.

## Asset References

These image references are available from the repository root:

```text
assets/pycharm-recovery-map.png
assets/lock-file-diagnosis.png
assets/interpreter-restoration-flow.png
```

From this `docs/` file, use:

```text
../assets/pycharm-recovery-map.png
../assets/lock-file-diagnosis.png
../assets/interpreter-restoration-flow.png
```
