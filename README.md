# cc-cli-launcher

A double-clickable macOS `.app` bundle that opens Terminal and runs `claude` (the Claude Code CLI). Lives in `/Applications` as **Claude Code Launcher** and can be pinned to the Dock.

This repo gives you the prompt, the build steps, and the troubleshooting notes. You bring your own icon (or generate one from a PNG using the recipe below) and your own preferences for working directory and bundle ID.

## Scope

This is a personal-use recipe. The build steps produce a launcher for your own machine and contain no Anthropic intellectual property in their default form.

You are responsible for what you put into the bundle. The CC0 dedication in this repository releases the code I wrote into the public domain. It does not, and cannot, affect Anthropic's trademarks (including "Claude" and "Claude Code"), the official Claude icon, or any other Anthropic brand asset. Embedding those in a build you share or distribute is a question between you and Anthropic.

Practical guidance: keep this bundle on your own Mac, source your own icon, and if you fork or republish the recipe, choose an app name that does not imply this is an Anthropic product.

## What's in this repo

- `cc-cli-launcher-prompt.md`: copy-paste prompt that drives Claude Code through the build for you.
- `README.md`: this file. Manual build steps, customization, and troubleshooting.
- `LICENSE`: CC0 1.0 Universal (public domain dedication).

## Bundle layout

The app bundle the prompt produces:

```
Claude Code Launcher.app/
  Contents/
    Info.plist                       metadata (name, identifier, icon ref)
    MacOS/
      ClaudeCodeLauncher             executable shell script (chmod +x)
    Resources/
      ClaudeCodeLauncher.icns        app icon (optional)
```

## How it works

When launched, `Contents/MacOS/ClaudeCodeLauncher` runs:

```bash
#!/bin/bash
osascript <<'EOF'
set wasRunning to application "Terminal" is running
tell application "Terminal"
    activate
    if wasRunning then
        do script "cd <YOUR_WORK_DIR> && claude"
    else
        repeat until (count of windows) > 0
            delay 0.05
        end repeat
        do script "cd <YOUR_WORK_DIR> && claude" in window 1
    end if
end tell
EOF
```

`osascript` tells Terminal to open a new window. Terminal spawns a login shell that sources `~/.zshrc` (or `~/.bash_profile`), so `claude` resolves through the user's normal `PATH` even though the launching process has a minimal environment.

The `wasRunning` branch handles a Terminal.app quirk: if Terminal isn't already running, `activate` launches it and Terminal creates its own default startup window, then `do script` would open a second window. The script captures Terminal's running state before activating it; when Terminal was not running, it reuses the auto-created startup window (`in window 1`) instead of opening a new one.

## Build it

Two options:

1. **Use Claude Code**: open `cc-cli-launcher-prompt.md`, paste its body into a Claude Code session, answer the three questions it asks (icon, working directory, bundle ID), and it builds the bundle for you.
2. **Manual**: see below.

### Manual build

Prerequisites: macOS, an admin user with write access to `/Applications`, the `claude` CLI installed and on your `PATH`, and a `.icns` icon file (or skip the icon; see "no icon" note below).

```bash
APP="/Applications/Claude Code Launcher.app"
ICON_SRC="/path/to/your/icon.icns"        # leave empty to skip the icon
WORK_DIR="$HOME"                          # directory the new Terminal cd's into
BUNDLE_ID="com.local.cc-cli-launcher"     # change if you want a unique LS registration

mkdir -p "$APP/Contents/MacOS" "$APP/Contents/Resources"

if [ -n "$ICON_SRC" ] && [ -f "$ICON_SRC" ]; then
    cp "$ICON_SRC" "$APP/Contents/Resources/ClaudeCodeLauncher.icns"
fi

cat > "$APP/Contents/Info.plist" <<PLIST
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleName</key>
    <string>Claude Code Launcher</string>
    <key>CFBundleDisplayName</key>
    <string>Claude Code Launcher</string>
    <key>CFBundleIdentifier</key>
    <string>${BUNDLE_ID}</string>
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
PLIST

cat > "$APP/Contents/MacOS/ClaudeCodeLauncher" <<SH
#!/bin/bash
osascript <<'EOF'
set wasRunning to application "Terminal" is running
tell application "Terminal"
    activate
    if wasRunning then
        do script "cd ${WORK_DIR} && claude"
    else
        repeat until (count of windows) > 0
            delay 0.05
        end repeat
        do script "cd ${WORK_DIR} && claude" in window 1
    end if
end tell
EOF
SH

chmod +x "$APP/Contents/MacOS/ClaudeCodeLauncher"
/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister -f "$APP"
```

If you skipped the icon, also remove the `CFBundleIconFile` key from the plist above so macOS doesn't look for a missing file.

### No icon

If you don't have a `.icns`, the app still works. macOS uses the generic application icon. You can add one later by dropping a `.icns` into `Contents/Resources/` and re-running `lsregister`.

### Build the icon from a PNG

Use artwork you own or have a licence to use. Do not source the PNG from Anthropic's brand assets.

Source PNG should be square. 1024x1024 is ideal; smaller sources work but will be soft at the largest icon size.

```bash
SRC="/path/to/source.png"
ICONSET="/tmp/icon.iconset"
OUT="/path/to/output/icon.icns"

rm -rf "$ICONSET"
mkdir -p "$ICONSET"
for spec in "16:icon_16x16.png" "32:icon_16x16@2x.png" "32:icon_32x32.png" "64:icon_32x32@2x.png" "128:icon_128x128.png" "256:icon_128x128@2x.png" "256:icon_256x256.png" "512:icon_256x256@2x.png" "512:icon_512x512.png" "1024:icon_512x512@2x.png"; do
  size="${spec%%:*}"
  name="${spec##*:}"
  sips -z "$size" "$size" "$SRC" --out "$ICONSET/$name" >/dev/null
done
iconutil -c icns "$ICONSET" -o "$OUT"
```

## Customization

Edit the launch script:

```bash
open -e "/Applications/Claude Code Launcher.app/Contents/MacOS/ClaudeCodeLauncher"
```

Common variations:

- **Change default directory**: replace whatever path comes after `cd` in the script.
- **Launch from current Finder location instead of a fixed dir**: drop the `cd` and use `do script "claude"`.
- **Pass flags**: e.g. `do script "cd $HOME && claude --resume"`.
- **Use iTerm2 instead**:
  ```applescript
  tell application "iTerm"
      activate
      create window with default profile command "claude"
  end tell
  ```
- **Skip Terminal entirely** (note: an interactive TUI like `claude` will not work usefully without a terminal):
  ```bash
  #!/bin/bash
  exec "$(which claude)"
  ```

After editing, re-run `chmod +x` if your editor stripped permissions, then test by clicking the app.

## Troubleshooting

**Dock shows generic icon after install:** `killall Dock`. If still wrong, also `killall Finder`.

**Clicking the Dock icon (or Spotlight result) does nothing, but the app works when opened from `/Applications`:** an old copy of the bundle with the same `CFBundleIdentifier` is still registered with LaunchServices, typically one sitting in the Trash. The Dock and Spotlight resolve the app by bundle ID and can land on the stale, launch-disabled copy. Check for duplicate registrations:
```bash
LSREGISTER=/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister
"$LSREGISTER" -dump | grep -B20 "^identifier: *com.local.cc-cli-launcher" | grep '^path:'
```
Unregister the stale path, re-register the real bundle, then restart the Dock:
```bash
"$LSREGISTER" -u "$HOME/.Trash/Claude Code Launcher.app"
"$LSREGISTER" -f "/Applications/Claude Code Launcher.app"
killall Dock
```
Substitute your own bundle ID and paths if you changed them. Emptying the Trash alone does not always clear the stale registration.

**App opens, Terminal opens, but `claude` not found:** the user's shell init (`~/.zshrc` or `~/.bash_profile`) is not putting the `claude` install directory on `PATH`. Verify `which claude` works in a fresh Terminal window. If it does, the app should also work; if it doesn't, fix the shell config first or replace `claude` in the launch script with the absolute path returned by `which claude`.

**App refuses to launch with "damaged" or signing error:** if you copy the bundle from a synced source (OneDrive, iCloud Drive, network share, an internet download), macOS may quarantine it.

**Executable bit lost after sync:** OneDrive and some other sync clients do not preserve the macOS executable bit. Restore it:
```bash
chmod +x "/Applications/Claude Code Launcher.app/Contents/MacOS/ClaudeCodeLauncher"
```

**Verify the script standalone:** running the executable directly shows any errors in your current terminal:
```bash
"/Applications/Claude Code Launcher.app/Contents/MacOS/ClaudeCodeLauncher"
```

## Licence

Released under [CC0 1.0 Universal](LICENSE), a public domain dedication. The dedication waives copyright in the code in this repository. It does not waive, and cannot waive, Anthropic's trademarks, logos, or other brand assets, which belong to Anthropic, PBC.

## Trademarks

Claude and Claude Code are trademarks of Anthropic, PBC. This project is an unofficial, personal-use launcher and is not affiliated with, endorsed by, or sponsored by Anthropic.

Users are responsible for sourcing their own icon. Do not embed Anthropic's official Claude icon or other brand assets in any build you intend to share or distribute. The recipe in [Build the icon from a PNG](#build-the-icon-from-a-png) is provided so you can generate one from artwork you have rights to use.
