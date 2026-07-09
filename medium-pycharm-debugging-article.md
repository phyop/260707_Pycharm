# Debugging PyCharm Startup Locks, Missing Conda Interpreters, and a Vanished CodeGlancePro Minimap on Windows

PyCharm did not fail in one clean, obvious way.

It started with a startup lock error. After that was fixed, every Python interpreter disappeared. After the interpreter list was restored, the editor minimap beside the right scrollbar was gone. The final fix was not one magic command, but a sequence of separating symptoms from root causes and restoring only the configuration that actually mattered.

This is the full debugging path.

![PyCharm recovery map](assets/pycharm-recovery-map.png)

## The First Error: PyCharm Claimed Another Instance Was Still Running

The first startup error looked like this:

```text
Internal error

com.intellij.platform.ide.bootstrap.DirectoryLock$CannotActivateException:
Process "C:\Program Files (x86)\JetBrains\PyCharm Community Edition 2022.3.2\bin\pycharm64.exe" (48464) is still running and does not respond.
```

The obvious first theory was:

```text
Maybe pycharm64.exe is really still alive and stuck.
```

So I checked for a running PyCharm process:

```powershell
Get-Process -Name pycharm64 -ErrorAction SilentlyContinue |
  Select-Object Id, ProcessName, Path, StartTime, Responding
```

I also searched for PyCharm in Windows Task Manager, but I could not find a visible PyCharm process.

That changed the interpretation of the error. The message mentioned a process ID, but the real blocker did not have to be a still-running window. JetBrains IDEs use lock and port marker files to detect whether another IDE instance owns the same configuration and system directories. A stale marker can outlive the process that created it.

At this point, I knew this was not going to be solved by simply killing `pycharm64.exe`.

## Then Reset Settings Failed Too

Trying to recover the IDE settings produced another error:

```text
Cannot write the reset marker file.

The cause: java.lang.AssertionError:
Marker file %APPDATA%\JetBrains\PyCharm2026.1\migrate.config shouldn't exist
```

This suggested a different failure mode:

```text
The settings migration or reset flow had started, but left behind a marker file.
```

After making sure PyCharm was closed, I removed the marker:

```powershell
Remove-Item -LiteralPath `
  "$env:APPDATA\JetBrains\PyCharm2026.1\migrate.config" `
  -Force
```

That fixed one layer, but PyCharm still did not start cleanly. This was the first important lesson: `migrate.config` was a symptom of a dirty recovery state, not the root cause of the startup lock.

## The Real Startup Blocker Was `.port`

The next error was more useful:

```text
java.nio.file.FileSystemException:
%LOCALAPPDATA%\JetBrains\PyCharm2026.1\.port:
The system cannot access the file.
```

The stacktrace included:

```text
Files.deleteIfExists
DirectoryLock.lockOrActivate
```

That was the clue. PyCharm was trying to delete `.port` during startup lock negotiation, and Windows refused access.

I checked both the local `.port` file and the roaming `.lock` file:

```powershell
Get-Item -LiteralPath `
  "$env:LOCALAPPDATA\JetBrains\PyCharm2026.1\.port" `
  -Force

Test-Path -LiteralPath `
  "$env:APPDATA\JetBrains\PyCharm2026.1\.lock"
```

The stacktrace also reported:

```text
NoSuchFileException:
%APPDATA%\JetBrains\PyCharm2026.1\.lock
```

So `.lock` was not the live blocker. `.port` was.

The fix was:

```powershell
Remove-Item -LiteralPath `
  "$env:LOCALAPPDATA\JetBrains\PyCharm2026.1\.port" `
  -Force
```

After removing `.port`, PyCharm finally opened.

![Lock file diagnosis](assets/lock-file-diagnosis.png)

## PyCharm Opened, but Every Interpreter Was Gone

Fixing startup uncovered the next problem: all Python interpreters were missing.

PyCharm prompted me to create:

```text
uv (example-project)
```

That interpreter was only a temporary response to the broken configuration. It was not the environment I actually needed. The original project used Conda, likely Python 3.11.

The new theory was:

```text
PyCharm recovered by creating a fresh settings state, while the old interpreter definitions were still in a JetBrains backup.
```

That turned out to be correct. JetBrains had left a backup directory:

```text
%APPDATA%\JetBrains\PyCharm2026.1-backup\2026-06-23-14-08
```

The global interpreter definitions live in:

```text
options\jdk.table.xml
```

I compared the current file with the backup:

```powershell
Get-Content -Raw `
  "$env:APPDATA\JetBrains\PyCharm2026.1\options\jdk.table.xml"

Get-Content -Raw `
  "$env:APPDATA\JetBrains\PyCharm2026.1-backup\2026-06-23-14-08\options\jdk.table.xml"
```

The current file mainly contained the newly created `uv` interpreter. The backup still contained the original interpreters, including entries like:

```text
Python 3.7 (python37)
Conda(python311)
Conda(MaskRCNN_TF2_Python3613)
Python 3.11
```

I restored the backup `jdk.table.xml`:

```powershell
Copy-Item `
  -LiteralPath "$env:APPDATA\JetBrains\PyCharm2026.1-backup\2026-06-23-14-08\options\jdk.table.xml" `
  -Destination "$env:APPDATA\JetBrains\PyCharm2026.1\options\jdk.table.xml" `
  -Force
```

That restored the global list of interpreters.

But the project also had to point back to the correct interpreter. The project-level setting was in:

```text
<project-root>\.idea\misc.xml
```

It had been changed to:

```xml
project-jdk-name="uv (example-project)"
```

I changed it back to:

```xml
project-jdk-name="Python 3.11"
```

The restored interpreter pointed to:

```text
<conda-install>\python.exe
Python 3.11.13
```

The key lesson here: restoring `jdk.table.xml` fixes what PyCharm knows globally, but `.idea\misc.xml` decides what the project actually uses.

![Interpreter restoration flow](assets/interpreter-restoration-flow.png)

## Removing the Unwanted `uv` Interpreter

Once the Conda interpreter was back, the temporary interpreter was no longer useful:

```text
uv (example-project)
```

I removed that entry from `jdk.table.xml`.

The order mattered. I did not remove `uv` first. I restored and verified the Conda interpreter first, then removed the temporary interpreter. That prevented PyCharm from falling back into another forced environment creation prompt.

## CodeGlancePro Was Gone Too

After startup and interpreter recovery, one UI feature was still missing: the code minimap beside the right scrollbar.

That was not a built-in PyCharm feature in this setup. It came from:

```text
CodeGlancePro
```

The next theory was:

```text
The settings reset did not only affect interpreters. It also affected plugins and plugin options.
```

The backup contained both the plugin and its settings:

```text
...\PyCharm2026.1-backup\2026-06-23-14-08\plugins\CodeGlancePro
...\PyCharm2026.1-backup\2026-06-23-14-08\options\CodeGlancePro.xml
```

I restored the plugin directory:

```powershell
Copy-Item `
  -LiteralPath "...\PyCharm2026.1-backup\2026-06-23-14-08\plugins\CodeGlancePro" `
  -Destination "$env:APPDATA\JetBrains\PyCharm2026.1\plugins\CodeGlancePro" `
  -Recurse `
  -Force
```

Then I restored the plugin options:

```powershell
Copy-Item `
  -LiteralPath "...\PyCharm2026.1-backup\2026-06-23-14-08\options\CodeGlancePro.xml" `
  -Destination "$env:APPDATA\JetBrains\PyCharm2026.1\options\CodeGlancePro.xml" `
  -Force
```

I also checked `disabled_plugins.txt` to make sure CodeGlancePro was not listed as disabled.

After restarting PyCharm, the minimap appeared again.

## The `.port` Error Returned Once More

After restoring plugins, the same `.port` startup error appeared again:

```text
%LOCALAPPDATA%\JetBrains\PyCharm2026.1\.port:
The system cannot access the file.
```

That confirmed `.port` was not just a cosmetic error message. It was part of PyCharm's startup lock state and could be recreated or left behind during a failed launch.

The same fix worked:

```powershell
Remove-Item -LiteralPath `
  "$env:LOCALAPPDATA\JetBrains\PyCharm2026.1\.port" `
  -Force
```

After that, PyCharm opened normally with the restored interpreter and CodeGlancePro enabled.

## The Final Recovery Sequence

The sequence that actually worked was:

1. Do not reinstall PyCharm immediately.
2. Check whether `pycharm64.exe` or the reported PID is still running.
3. Remove the stale `migrate.config` marker.
4. Treat `.port` as the real startup lock blocker.
5. Remove `Local\JetBrains\PyCharm2026.1\.port`.
6. Start PyCharm and confirm it opens.
7. Restore `jdk.table.xml` from the JetBrains backup.
8. Point `.idea\misc.xml` back to the Conda/Python 3.11 interpreter.
9. Remove the temporary `uv (example-project)` interpreter.
10. Restore CodeGlancePro and its options from the backup.
11. If `.port` returns after another failed launch, remove it again after PyCharm is closed.

## What I Learned

`DirectoryLock$CannotActivateException` does not always mean there is a visible PyCharm process to kill. It can also mean JetBrains startup lock state has gone stale.

`migrate.config` and `.port` are different layers of failure. The migration marker explained why reset failed. The `.port` file explained why startup still failed.

Interpreter recovery is not the same as environment recreation. In this case, the Conda environment was still there. PyCharm had simply lost its interpreter definitions.

Project interpreter selection has two layers: global interpreter definitions in `jdk.table.xml`, and project selection in `.idea\misc.xml`.

Plugins can be affected by the same settings reset. If an editor feature suddenly disappears, check the plugin folder, plugin options, and `disabled_plugins.txt` before assuming the IDE feature was removed.

## Final State

The recovered setup was:

```text
PyCharm opens successfully
Project interpreter: Python 3.11 / Conda
Conda path: <conda-install>\python.exe
Temporary uv interpreter removed
CodeGlancePro restored
Editor minimap visible again
Stale .port startup lock cleared
```

This was a reminder that IDE recovery is not just about deleting caches. The useful work is identifying which error is a symptom, which file is the blocker, and which parts of the old configuration are worth restoring.
