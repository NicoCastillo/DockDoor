# Fork Notes

Personal notes for this fork (`NicoCastillo/DockDoor`). Not part of upstream.

## Setting up on a new machine

### 1. Build

There is no signing certificate configured, so build ad-hoc signed:

```bash
xcodebuild -project DockDoor.xcodeproj -scheme DockDoor -configuration Release \
  -destination 'platform=macOS' \
  CODE_SIGN_IDENTITY="-" CODE_SIGNING_REQUIRED=NO CODE_SIGNING_ALLOWED=NO DEVELOPMENT_TEAM="" \
  build
```

The app lands in:

```
~/Library/Developer/Xcode/DerivedData/DockDoor-*/Build/Products/Release/DockDoor.app
```

Install it by quitting DockDoor, then `ditto`ing that bundle over `/Applications/DockDoor.app`.

### 2. Turn on the folder widget — easy to forget

**Upstream ships `enableFolderWidget` defaulting to `false`.** Hovering a folder in the
dock (Downloads, etc.) will show *nothing at all* until it is switched on, which looks
exactly like the feature being broken.

Quit DockDoor first, or the running app overwrites the value on exit:

```bash
osascript -e 'tell application "DockDoor" to quit'
defaults write com.ethanbills.DockDoor enableFolderWidget -bool true
open -a /Applications/DockDoor.app
```

Or in the GUI: **Settings → Dock Previews → enable the folder widget.** (It used to live
under Widgets; upstream moved it in `c60ec54`.)

Setting it explicitly also pins it, so a future upstream default change can't flip it back.

### 3. Repeat per macOS user account

Preferences are per-user. Each account that uses DockDoor needs its own
`enableFolderWidget` flip, even though `/Applications/DockDoor.app` is shared.

For another account, easiest is to toggle it in the GUI while logged in as them. From an
admin session, run it *as* that user so the plist stays owned by them:

```bash
sudo -u <username> defaults write com.ethanbills.DockDoor enableFolderWidget -bool true
```

### 4. Re-grant Accessibility after every install

Accessibility is a per-user TCC grant keyed to the binary's code hash. Replacing an
ad-hoc signed build invalidates it, so after installing, previews may not appear at all —
apps included, not just folders. Fix in **System Settings → Privacy & Security →
Accessibility**: toggle DockDoor off, then on.

If *only* folder previews are missing, that's step 2, not this.

## What this fork changes

- `5b5b9a0` — folder widget file rows are draggable to other apps / Finder
  (`FolderWidgetView.swift`)
- `56b2074` — folder widget preview stays anchored when the cursor moves onto it, instead
  of shifting toward the dock (`SharedPreviewWindowCoordinator.swift`)

The anchor fix is in otherwise-unmodified upstream code, so it is a candidate to submit
upstream. Doing so would remove the need to carry it across syncs.

## Syncing upstream

```bash
git fetch upstream
git merge-tree --write-tree HEAD upstream/main >/dev/null && echo "clean merge"
git merge upstream/main
```

Merge rather than rebase, to avoid rewriting the commits above. After merging, rebuild and
reinstall (steps 1 and 4), and re-check that upstream hasn't changed any defaults you rely
on:

```bash
git diff <old> <new> -- DockDoor/consts.swift
```

Default changes are silent and only affect keys you never set explicitly — that is exactly
how the folder widget disappeared during the 1.39.4 sync.
