# ThemePicker

Pick one theme, sync it to nvim, kitty, ghostty, waybar, rofi, tmux, and GTK.

```
theme-picker <name>     apply a theme
theme-picker list        list available themes (* = current)
theme-picker current     print the current theme's name
theme-picker rofi        pick a theme from a rofi menu and apply it
```

`theme-picker` is symlinked onto `PATH` at `~/.local/bin/theme-picker`, pointing at
this repo's `theme-picker` script — edit that script in place, no reinstall needed.

## Themes

Each theme lives in `themes/<slug>/` as three files:

- `kitty.conf` — a standard kitty color-scheme file (background, foreground,
  cursor, `color0`-`color15`, ...).
- `ghostty.conf` — the same palette in Ghostty's theme-file format
  (`background =`, `palette = N=#hex`, ...).
- `waybar-colors.css` — eight semantic GTK `@define-color` vars: `base`,
  `text`, `accent`, `surface`, `red`, `muted-bg`, `muted-fg`, `overlay`.
- `rofi-colors.rasi` — the same palette in the adi1090x rofi-collection's
  `colors/*.rasi` format: `background`, `background-alt`, `foreground`,
  `selected`, `active`, `urgent`.
- `tmux.conf` — the same semantic colors as `waybar-colors.css`, as
  `set -g @minimal-tmux-*` options for the `minimal-tmux-status`-derived
  status line: `@minimal-tmux-bg`, `@minimal-tmux-fg`,
  `@minimal-tmux-accent-subtle`, `@minimal-tmux-accent-text`,
  `@minimal-tmux-accent-indicator`, `@minimal-tmux-accent-session`.
- `gtk-colors.css` — the same eight semantic colors, remapped as
  `@define-color` overrides under both the classic GTK3 names (`theme_bg_color`,
  `theme_selected_bg_color`, `borders`, ...) and the libadwaita/GTK4 names
  (`window_bg_color`, `accent_color`, `card_bg_color`, ...), so it overrides
  whichever set the active GTK theme actually reads. Colors this palette has
  no semantic for (warning/success hues, etc.) are left alone.

Current set: `gruber-darker`, `catppuccin-mocha`, `tokyo-night`, `nord`, `onedark`,
`rose-pine`, `gruvbox`.

### Adding a theme

Add a new `themes/<slug>/` directory with those six files, plus an entry
in the `THEMES` table in
`~/.config/nvim/lua/mulabeg/plugins/theme-picker.lua` (repo, plugin `name`,
`:colorscheme` name, and an optional `setup()` for plugins that need
configuring before their colorscheme is applied — see the `onedark` entry).
`theme-picker list` picks up the new slug automatically once the directory
exists.

## How syncing works, per app

- **kitty** — `theme-picker` copies `themes/<slug>/kitty.conf` to
  `~/.config/kitty/theme.conf`, which `kitty.conf` `include`s. It then sends
  `kitty` `SIGUSR1`, which makes it reload its config (including includes)
  without restarting — so this is instant for open kitty windows.

- **ghostty** — the palette is copied to `~/.config/ghostty/themes/<slug>`
  (a custom theme file, same mechanism as the existing `dankcolors` theme)
  and `~/.config/ghostty/config`'s `theme = ` line is rewritten to point at
  it. **Ghostty does not watch its config file** — already-open windows need
  a manual reload (default keybind `ctrl+shift+,` on Linux); new windows
  pick up the change on their own.

- **waybar** — the semantic vars are written to `~/.config/waybar/colors.css`,
  which `style.css` `@import`s (the rest of `style.css` was refactored to
  reference `@base`/`@text`/`@accent`/etc. instead of hardcoded hex). The
  script then kills waybar (`pkill -x waybar`, waiting for it to exit) and
  relaunches it detached (`setsid waybar & disown`) so the new colors take
  effect immediately.

- **rofi** — `themes/<slug>/rofi-colors.rasi` is copied to
  `~/.config/rofi/colors/current.rasi`. Every launcher/applet/powermenu style
  in `~/.config/rofi/{launchers,applets,powermenu}/**/shared/colors.rasi`
  normally `@import`s one hardcoded palette from `~/.config/rofi/colors/`
  (mostly `onedark.rasi`); the script rewrites all of those `@import` lines
  to point at `current.rasi` instead, the same include-rewrite trick used
  for kitty/ghostty. rofi has no daemon here — each launch reads the theme
  files fresh, so nothing needs restarting.

  `theme-picker rofi` opens a rofi **script-mode** menu (`theme-picker-rofi-script`,
  which just execs `theme-picker --rofi-script`) listing the themes (marking
  the current one), styled with whichever applet style is currently set in
  `~/.config/rofi/applets/shared/theme.bash`. Selecting a theme (Enter)
  applies it for real and saves it to `~/.config/theme-picker/current`, same
  as `theme-picker <name>`; Esc closes the menu without changing anything.

- **tmux** — `themes/<slug>/tmux.conf` is copied to `~/.config/tmux/theme.conf`,
  which `~/.config/tmux/tmux.conf` `source-file`s right before it `run`s
  `~/.config/tmux/minimal.tmux` (a parameterized fork of the
  `minimal-tmux-status` plugin that reads the `@minimal-tmux-*` options
  set there). tmux has no config-file watcher, so the script reloads the
  running server itself (`tmux source-file ~/.config/tmux/tmux.conf`),
  same as kitty's SIGUSR1 reload — no detach/reattach needed.

- **gtk** — `themes/<slug>/gtk-colors.css` is copied to
  `~/.config/gtk-{3.0,4.0}/colors.css`, and each directory's `gtk.css` is made
  to `@import` it last (so it wins the cascade). What happens to the rest of
  `gtk.css` depends on what was already there:
  - missing — a fresh `gtk.css` is written with just the `@import`.
  - a symlink (common for `gtk-4.0/gtk.css`, since GTK4 has no
    `gtk-theme-name` lookup of its own — theme installers point this file at
    a theme's own stylesheet directly) — the real target is captured once to
    `~/.config/theme-picker/gtk4-base-css`, the symlink is replaced with a
    real file, and that target is `@import`ed first so the rest of the theme
    (chrome, icons, borders, ...) is preserved, our colors layered on top.
  - an existing regular file (a theme's own CSS, or hand-written overrides)
    — left untouched, with the `@import` appended at the end.

  GTK watches its own `gtk.css` and reapplies changes live for most
  GTK4/libadwaita apps; GTK3 apps generally need a restart to re-read it.
  Only the eight semantic colors are overridden (background/text/accent/
  surface/red/muted/overlay, under both the classic GTK3 names and the
  libadwaita ones) — anything the palette has no slot for (warning/success
  hues, icon theme, border radii, ...) is left to the underlying theme.

- **nvim** — the chosen slug is written to `~/.config/theme-picker/current`.
  `~/.config/nvim/lua/mulabeg/plugins/theme-picker.lua` reads that file at
  startup to decide which colorscheme plugin to load eagerly, and also
  watches the file (same `fs_event` pattern as the existing
  `plugins/dankcolors.lua`) so already-open nvim instances switch live when
  you run `theme-picker` from a shell.

## Not yet wired up

Chromium — deliberately out of scope for now. Chromium has no config-file
theming; doing this properly means a small unpacked extension using
`chrome.theme.update()`, fed live color data some other way (native
messaging host, or the extension polling a local file). Worth its own pass
later if wanted.
# ThemePicker
