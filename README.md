# Fixing PowerShell startup overhead in the Windows Codex app

We reduced avoidable shell startup overhead and corrected which PowerShell installation the app uses. The final verified setup is **PowerShell 7.6.6, 64-bit, starting with `-NoProfile` automatically**.

The largest demonstrated gain came from skipping profile scripts. Selecting the newer 64-bit installation produced a smaller measured startup improvement. This did not make every command, build, or app interaction faster.

## The problems and how we fixed them

### 1. Every command could load interactive profile customizations

PowerShell profiles loaded Terminal-Icons even though routine agent commands did not need it. Under the sandbox, that module also attempted cache writes that were denied.

After backing up the existing Codex configuration, we added this top-level setting to `~/.codex/config.toml`:

```toml
# Skip shell profiles for fast, predictable agent commands.
allow_login_shell = false
```

This makes omitted `login` arguments default to non-login shells and rejects explicit `login = true` requests. In this Windows setup, that results in PowerShell starting with `-NoProfile`.

The running app initially kept using its loaded defaults. Until restart, commands explicitly used `login: false`. After restart, we checked a normal command with no shell or login override: the actual launch arguments included `-NoProfile`, and Terminal-Icons was not loaded.

We did not edit the interactive PowerShell profile, weaken the sandbox, change execution policy, or disable antivirus.

### 2. Installing the newer PowerShell did not make the app select it

Both versions were installed:

| Installation | Location |
|---|---|
| PowerShell 7.6.5, 32-bit | `C:\Program Files (x86)\PowerShell\7\pwsh.exe` |
| PowerShell 7.6.6, 64-bit | `C:\Program Files\PowerShell\7\pwsh.exe` |

Windows' machine PATH listed the old installation first. PATH is the ordered list of folders Windows searches for executable commands.

Supplying the newer executable through the tool's shell option did not force that exact file. The public Codex source inspected during the investigation used that option to identify a shell type, then discovered the executable through its search logic. Calling the newer executable directly worked, but wrapping every command in another shell would add unnecessary startup overhead.

We backed up the machine PATH and prepared a change that moved the existing 64-bit PowerShell directory ahead of the old installation while preserving the other entries and registry value type.

The first registry write was denied because it required administrator rights. We then used a narrowly scoped script through Windows UAC approval. The script applied the prepared change, verified the saved value, and notified Windows of the environment change. Neither PowerShell installation was removed.

After fully restarting the app, a normal agent command verified:

```text
Version:              7.6.6
Architecture:         X64
Executable:           C:\Program Files\PowerShell\7\pwsh.exe
Launch arguments:     -NoProfile -Command
Terminal-Icons loaded: False
```

The version-selection problem is resolved. No further restart was needed after this verification.

### 3. Many small tool calls add up

A fast command still has shell and tool-call overhead. We compared three reads of the same local file through separate calls against the same reads batched into one call:

| Approach | Measured tool round-trip time |
|---|---:|
| Three separate calls | 1,952 ms total |
| Three reads in one call | 664 ms |
| Difference | **1,288 ms saved, about 66%** |

This was one small comparison, not a broad benchmark. Batching was already required by the existing instructions; it was not a new persistent configuration change. It demonstrates why unnecessary separate calls can make work feel slow.

PowerShell 7 and ripgrep (`rg`) were already installed. We did not need more search tools or a move to Linux for these improvements.

## Measured performance

### The main improvement: skip profiles

A five-run comparison under the sandbox produced these startup medians:

| Startup mode | Median |
|---|---:|
| Profiles loaded | 652 ms |
| Profiles skipped | 218 ms |
| Improvement | **About 3× faster; 67% less startup time** |

The profile-enabled runs included Terminal-Icons permission errors. These results describe this machine and configuration, not PowerShell performance in general.

### Final results after selecting 64-bit PowerShell

| Measurement | Earlier no-profile 7.6.5 X86 | Final no-profile 7.6.6 X64 | Observed change |
|---|---:|---:|---|
| Shell startup | 213.9 ms | **193.4 ms** | About 9.6% lower |
| Read AGENTS.md | 1.05 ms | **0.95 ms** | 0.10 ms lower |
| Search AGENTS.md with rg | 92 ms | **107.1 ms** | 15.1 ms higher |
| Trivial complete tool call | 483 ms | **493 ms** | 10 ms higher |

Startup measurements used seven launches per comparison; file reads and searches used five runs; trivial tool calls used three. Values are medians. The read/search baselines came from an earlier check than the startup/tool-call baselines. Final startup ranged from 188.1 to 229.4 ms.

File reads and searches were timed inside an already-running shell. Tool round trips were timed around the tool invocation, excluding the model's time thinking between calls and the user's full UI wait.

These were small sequential local checks, subject to machine load, caching, and normal variation. They do not isolate version changes from architecture changes or establish that either caused every difference. Whole tool calls did not improve in the final sample, and search was slightly slower.

Comparing the earlier profile-enabled 652 ms with the final 193.4 ms gives approximately 70% less startup time. That combines different configurations and runs; it must not be presented as a 70% speedup for the entire app.

## The final AGENTS.md instructions

We initially added too much text, then shortened the rules while preserving the important safeguards. This is the installed PowerShell section:

```markdown
## PowerShell execution
- Use PowerShell 7 with `login: false` for routine commands; load profiles only when needed. For direct launches, use `pwsh -NoLogo -NoProfile -NonInteractive`.
- Avoid nested shells; use the tool's working-directory parameter. Single-quote literal strings and use `-LiteralPath` where supported. Backslash does not escape PowerShell quotes. Pass native arguments separately; never use `Invoke-Expression` or build executable command strings. Prefer `.ps1` files for complex logic.
- Distinguish shell startup, command runtime, and tool latency when diagnosing slowness. Do not repeatedly rerun stalled commands or weaken security protections for speed.
```

The configuration is stricter than the instruction's profile exception: `allow_login_shell = false` prevents an agent from simply requesting `login: true`. A future need for profile loading would require deliberately revisiting that setting.

Existing rules already required scoped searches, batching independent reads, and using `rg`. We kept those rather than adding duplicate instructions.

## Safer Windows file operations

Screenshots alleging a destructive cross-shell command also prompted a review of file-operation rules. We did not independently verify the incident shown in those screenshots.

We strengthened the existing instruction:

```markdown
- Prefer patch/edit tools; use UTF-8 for generated text. Before recursive deletion/moves, inspect resolved absolute targets and verify containment within the authorized directory using a path-separator boundary. Reject empty targets, drive roots, and unexpected junctions/symlinks. Use native PowerShell with `-LiteralPath`, never cross-shell deletion/move commands. Preview recursive deletions with `-WhatIf`; previews do not replace authorization. Never bypass a blocked action.
```

This reduces reliance on fragile quoting across multiple shells. Instructions and `-WhatIf` do not replace authorization, sandbox boundaries, or backups. Backups were retained before the instruction, configuration, and PATH edits.

## Final status and limits

The app now selects the installed 64-bit PowerShell and skips profiles automatically. Both were verified in the actual agent process after restart, and performance was measured afterward.

An earlier claim that everything was resolved was too broad. A later report of continued slowness did not reproduce a long stall in these checks. The verified fixes address startup overhead and executable selection; they do not explain every possible delay.

Builds, downloads, broad searches, model response time, and app/tool overhead remain separate costs. If another long delay appears, the useful next evidence is the specific command, its execution duration, error output, and the wait before execution. More settings or tools should not be added without that evidence.

Subagents using the same configuration should inherit its startup behavior, but no separate subagent benchmark was performed. Delegation has its own overhead.

## Sources

- [Official Codex configuration reference](https://developers.openai.com/codex/config-reference/) — `allow_login_shell` behavior.
- [Official Windows app documentation](https://developers.openai.com/codex/app/windows) — native Windows support and sandbox guidance.
- [Microsoft: PowerShell profiles](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_profiles) — starting without profiles.
- [Microsoft: PowerShell quoting rules](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_quoting_rules) — quoting and native-command considerations.
- [Codex shell-selection source](https://github.com/openai/codex/blob/main/codex-rs/shell-command/src/shell_detect.rs) — executable discovery; the linked main branch can change.
- [PowerShell 7.6.6 release notes](https://github.com/PowerShell/PowerShell/releases/tag/v7.6.6) — no general startup-speed improvement was listed.

Measured October 10–11, 2026. Finalized October 11, 2026. Results are from one Windows machine, not general performance guarantees.
