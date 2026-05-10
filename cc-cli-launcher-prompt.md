# cc-cli-launcher Prompt (macOS)

> **Personal use only.** This prompt produces a launcher for your own Mac. Claude and Claude Code are trademarks of Anthropic, PBC. This prompt is not affiliated with, endorsed by, or sponsored by Anthropic. Do not embed Anthropic's official Claude icon or other brand assets in any build you intend to share or distribute.
>
> **Licence.** This prompt is dedicated to the public domain under [CC0 1.0 Universal](LICENSE).

A reusable Claude Code prompt that builds a double-clickable macOS `.app` bundle which opens Terminal and runs `claude` (the Claude Code CLI). The built app is installed at `/Applications/Claude Code Launcher.app`.

Copy everything between the `===` markers below into a Claude Code session. The agent will ask you for the icon source and the default working directory before it builds anything.

===

I want a double-clickable macOS `.app` bundle that opens Terminal and runs `claude` (the Claude Code CLI). Build it directly as a hand-crafted bundle (no Automator). Use these exact specs.

**Target path:** `/Applications/Claude Code Launcher.app`

**Before you start, ask me for:**

1. **Icon source path**: the absolute path to a `.icns` file you should use as the app icon. If I do not have one, ask whether I want to proceed without an icon (the app will fall back to the generic macOS app icon) or whether I want to generate one from a PNG using `sips` + `iconutil` first. Do not assume a path. Do not pick one for me. The artwork must be something I own or have a licence to use; do not source the PNG from Anthropic's brand assets.
2. **Default working directory**: the directory the new Terminal window should `cd` into before launching `claude`. Offer `$HOME` as a sensible default if I have no preference, and confirm whether to drop the `cd` entirely if I want `claude` to launch in whatever directory Terminal opens in.
3. **Bundle identifier**: the reverse-DNS bundle ID to use in `Info.plist`. Suggest `com.local.cc-cli-launcher` as a safe default for personal use. Do not bake in a company or organization identifier unless I give you one.

**Bundle layout:**

```
Claude Code Launcher.app/
  Contents/
    Info.plist
    MacOS/ClaudeCodeLauncher           (executable shell script)
    Resources/ClaudeCodeLauncher.icns  (copied from icon source, if provided)
```

**Info.plist contents** (substitute `<BUNDLE_ID>` with the value I gave you):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleName</key>
    <string>Claude Code Launcher</string>
    <key>CFBundleDisplayName</key>
    <string>Claude Code Launcher</string>
    <key>CFBundleIdentifier</key>
    <string><BUNDLE_ID></string>
    <key>CFBundleVersion</key>
    <string>1.0</string>
    <key>CFBundleShortVersionString</key>
    <string>1.0</string>
    <key>CFBundlePackageType</key>
    <string>APPL</string>
    <key>CFBundleExecutable</key>
    <string>ClaudeCodeLauncher</string>
    <key>CFBundleIconFile</key>
    <string>ClaudeCodeLauncher</string>
    <key>LSMinimumSystemVersion</key>
    <string>10.13</string>
    <key>NSHighResolutionCapable</key>
    <true/>
</dict>
</plist>
```

If I declined to supply an icon, omit the `CFBundleIconFile` key (do not leave a dangling reference).

**Executable script (`Contents/MacOS/ClaudeCodeLauncher`)**: substitute `<WORK_DIR>` with the directory I gave you. If I asked you to drop the `cd`, omit it and just run `claude`:

```bash
#!/bin/bash
osascript <<'EOF'
tell application "Terminal"
    activate
    do script "cd <WORK_DIR> && claude"
end tell
EOF
```

**Steps you must take, in order:**

1. Ask me the three questions above (icon source, working directory, bundle ID). Do not proceed until you have answers.
2. Verify `/Applications` is writable. If not, abort and ask me.
3. If I provided an icon path, verify it exists and is a `.icns` file. If not, abort and ask me.
4. Verify `/Applications/Claude Code Launcher.app` does NOT already exist. If it does, abort and ask me before overwriting.
5. Resolve `claude` on my `PATH` (`which claude`) and confirm the binary exists. Note the path in your output. Do NOT bake that absolute path into the script. Terminal's login shell will resolve `claude` from the user's shell config.
6. Create the directory structure.
7. If I provided an icon, copy it to `Contents/Resources/ClaudeCodeLauncher.icns`.
8. Write `Info.plist` with the contents above (with the bundle ID and icon key handled correctly).
9. Write the executable script with the contents above (with the working directory substituted).
10. `chmod +x Contents/MacOS/ClaudeCodeLauncher`.
11. Register with Launch Services so Finder/Dock pick up the icon: `lsregister -f` on the bundle.
12. Test: launch the app with `open`, wait a few seconds, then query Terminal via `osascript` to confirm a new window exists with a process tree containing `claude`. Report the result.

Do NOT use Automator. Do NOT use `sudo`. Confirm with me before any destructive step.

If everything works, tell me the bundle is ready and remind me I can drag it to the Dock.

===

## Notes

- The bundle ID matters for Launch Services. If you build a second variant (e.g., a "Claude Code Launcher Beta" pointing at a different binary), give it a different bundle ID so macOS treats them as distinct apps.
- Do not collide with `/Applications/Claude.app` (Anthropic's official Claude desktop app). Always use the name `Claude Code Launcher.app`.
- The launch script uses Terminal's `do script`, which spawns a login shell. That shell sources the user's shell config (`~/.zshrc` or `~/.bash_profile`) and inherits the full `PATH`, which is why `claude` resolves without an absolute path. If the shell config does not put the directory containing `claude` on `PATH`, the new Terminal window will print "command not found" and exit. Fix it by repairing the shell config, or by hard-coding the absolute path returned by `which claude` into the launch script.
- To regenerate an `.icns` from a PNG: scale the source to the standard sizes (16, 32, 64, 128, 256, 512, 1024) with `sips`, drop them into a `.iconset` folder following Apple's naming convention, then run `iconutil -c icns Foo.iconset`. Use artwork you own or have a licence to use.

## How to use this prompt non-interactively

If you want the agent to execute without confirmation pauses (e.g., a batch script setting up a new machine), supply the icon path, working directory, and bundle ID up front in your initial message and prepend:

> Skip confirmation prompts and proceed directly through all steps. Only stop if a precondition genuinely fails.

For first-run rebuilds, leave the confirmations in. They catch real issues (e.g., a pre-existing `Claude.app` collision, a missing icon file, a `PATH` that does not include `claude`).
