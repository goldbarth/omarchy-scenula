# Scenula release and migration notes

## Lifecycle findings

Checked against the installed Omarchy plugin commands and `shell/services/PluginRegistry.qml` on 2026-10-30.

- `omarchy-plugin-add` clones to a staging directory, validates, reads the manifest ID, rejects that ID if already installed, then moves the clone into `~/.config/omarchy/plugins/<id>`.
- `omarchy-plugin-update` fetches `origin HEAD`, fast-forwards in place, validates, and rescans the shell.
  It has no install/update migration hook and does not rename checkouts or rewrite `shell.json` for a changed ID.
- The registry scans each installed directory's root `manifest.json` and keys plugins by manifest ID.
  The directory name does not have to match the ID for discovery.
  Canonical widget aliases are host-defined; there is no third-party rename declaration that carries forward enabled state.
- Third-party enabled state comes from exact ID references in `bar.layout`, `plugins`, and `disabledPlugins`.
  Consequently a root manifest ID change prevents the new service from loading under an old bar entry.
  A migration implemented only in service startup cannot repair this.
- Update/removal commands locate checkouts by directory name.
  The checkout must move together with the shell references to support subsequent updates/removal using the new ID.

Changing the ID inside the Tessella repository would therefore break the widget for every user on an ordinary update.
A single repository also cannot serve both IDs: `omarchy-plugin-update` always follows `origin HEAD`.
Scenula therefore ships from its own repository, and Tessella gets a final 0.2.1 bridge release with the old ID.
The bridge shows a rename banner, a daily notification and a **Migrate to Scenula** button.
The button installs Scenula next to Tessella (not enabled) and runs `scenula.py migrate`, see [README.md](README.md#upgrading-from-tessella).
Users who never update Tessella keep a working 0.2.x install.

## Storage strategy

Fresh installs create native Scenula directories.
Migrated installs create new Scenula symlinks to the existing Tessella config, data and state directories.
Legacy directories are the active backing storage and are never deleted automatically.
No library schema, tag version, swatch version, download URL or selection algorithm changes accompany this rename.
Absolute library and state paths, the active background symlink, and in-flight render/sync locks remain valid.
The migration does not perform any image processing or network requests.
Normal daily sync and rotation retain their existing behavior after startup.

The shell config receives a byte-for-byte backup before changing identity fields, and updates atomically with its original permissions.
A symlinked shell config remains a symlink.
Conflicting independent data directories, both plugins enabled, and a Tessella checkout with local changes are refused rather than merged, deleted or hidden.
The Tessella checkout is removed only as the last step, so an interrupted migration can be rerun with the same command.
The old IPC target routes to the same handler code once the Tessella widget is gone.
Absolute paths in external scripts to the old plugin checkout require manual adjustment.

## Publication steps

1. Create `goldbarth/omarchy-scenula` as a new public repository and push this history.
   Do not rename `goldbarth/omarchy-tessella`: existing checkouts must keep fetching the bridge release from it.
2. Merge the Tessella revert (#15) and the 0.2.1 bridge release in `goldbarth/omarchy-tessella`, only after step 1.
   The bridge installs from the Scenula repository, so it must exist first.
3. Marketplace: request the Tessella 0.2.1 update through the normal verification workflow; its plugin ID set is unchanged.
   Submit Scenula as a new listing.
   After the transition period, ask the maintainers to delist Tessella and archive its repository.
4. After publication, run the bridge once from a real Tessella install: update, open the panel, choose **Migrate to Scenula**.

## Validation

```bash
python3 -m unittest discover -s tests -v
SCENULA_INTEGRATION=1 python3 -m unittest discover -s tests -v
SCENULA_INTEGRATION=1 SCENULA_QPA_PLATFORM=wayland python3 -m unittest discover -s tests -v
omarchy plugin validate .
git diff --check
```

Integration tests require the installed Omarchy tools, Quickshell and ImageMagick.
They run with a temporary HOME and isolated Quickshell instance using the installed registry and UI types.
The default offscreen backend verifies service/widget loading and IPC; a Wayland run also verifies the real panel can instantiate, with panels kept closed.
Git installation/update, manifest discovery, widget/service loading, IPC and ImageMagick rendering run for real.
Only monitor detection and the final wallpaper setter are replaced, so they cannot change the real desktop wallpaper.
The upgrade fixture starts with legacy identity/configuration and populated images, metadata, collages and state.
The tests install Scenula next to a running Tessella widget, then verify migration, legacy/new IPC, automatic rotation, restart persistence and subsequent updates.
Offline regressions cover idempotency, concurrent initialization, conflict refusal, local changes, malformed configuration, shell config symlinks, lock sharing, cancellation and data integrity.
Visible compositor placement and the live GitHub install still need the post-publication check above.

## Results on 2026-10-31

Validated on Omarchy 4.0.6-1 and Quickshell 0.3.2.r7.g2b4501a-1.
All 21 tests passed with the offscreen and Wayland backends, including real panel instantiation with closed windows.
An end-to-end run in a temporary HOME installed the real Tessella 0.2.1 bridge with `omarchy plugin add --enable`, then ran its real `migrate-to-scenula.sh` against a local copy of this repository.
Result: Scenula took over the bar entry, settings were kept, the Tessella checkout was removed, and both IPC targets answered.
Manifest validation and `git diff --check` passed.
The real installation and its backing data were not migrated during validation.
