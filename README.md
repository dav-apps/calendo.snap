# Calendo Snap

A thin native window around the [Calendo](https://calendo.dav-apps.tech) web app.

The point of this package is that it does **not** bundle a browser. WebKitGTK
comes from the `gnome-46-2404` platform snap that the `gnome` extension pulls
in, so the download stays in the tens of kilobytes.

## Layout

```
snap/snapcraft.yaml               packaging
snap/gui/calendo.desktop          desktop entry (en + de)
snap/gui/calendo-notifier.desktop autostart entry for the reminder daemon
snap/gui/calendo.png              512px app icon, see below
src/calendo                       the window (Python + GTK3 + WebKit2 4.1)
src/calendo-notifier              the reminder daemon (see Reminders below)
```

## Icon

`snap/gui/calendo.png` is `assets/icons/icon-512x512-maskable.png` from the
PWA manifest — the maskable variant, because its logo sits in a safe zone and
so keeps more room to the edge than the regular icon. Maskable icons are
full-bleed squares, since the platform is meant to punch the shape out
itself; GNOME does not, so the corners are rounded here instead.

The radius matches the regular icon's tile: that one is 406x406 inside the
512px canvas with a 30px radius, i.e. 7.4%, which is 38px at full bleed. To
regenerate after an icon change:

```python
from PIL import Image, ImageDraw
im = Image.open("icon-512x512-maskable.png").convert("RGBA")
w, h, ss = *im.size, 4
r = round(30 / 406 * w)
mask = Image.new("L", (w * ss, h * ss), 0)
ImageDraw.Draw(mask).rounded_rectangle([0, 0, w * ss - 1, h * ss - 1],
                                       radius=r * ss, fill=255)
im.putalpha(mask.resize((w, h), Image.LANCZOS))
im.save("snap/gui/calendo.png")
```

Note that GNOME Shell caches icons by path, and the path never changes
(`/snap/calendo/current/meta/gui/calendo.png`). After an icon change users
see the new one only once they log in again, unless the file is renamed.

## Build

The host must build in a managed instance — a core24 snap cannot be built
destructively on a non-24.04 host.

```sh
snapcraft pack --use-lxd
sudo snap install --dangerous ./calendo_1.0.0_amd64.snap
```

`--dangerous` is needed because a locally built snap carries no store
signature. To remove it again: `sudo snap remove calendo`.

Run from a terminal to see errors:

```sh
snap run calendo
CALENDO_DEBUG=1 snap run calendo   # enables the WebKit inspector (right click)
```

## Where data lives

Everything the web app stores — localStorage, IndexedDB, service workers,
cookies — is kept under:

```
~/snap/calendo/common/profile/
```

`common` rather than the revisioned directory, so a snap refresh does not
copy the profile around. This store is **separate** from the browser's: the
app has its own session and needs its own dav login. Syncing happens through
the dav account, not through shared local files.

## Publishing

```sh
snapcraft login
snapcraft register calendo        # once, the name is global
snapcraft upload --release=edge ./calendo_1.0.0_amd64.snap
```

arm64 has no local build here; `snapcraft remote-build` covers it. The part
only copies files, so nothing is architecture specific.

## Reminders

WebKitGTK has no Push API — the machinery is compiled in but no public API is
exported and the platform snap ships no push daemon. Web push is therefore
out of reach, and it is also unnecessary.

A dav notification is a *scheduled object* (`time`, `interval`, `title`,
`body`) that `listNotifications` hands out over the API. Web push is only the
transport a browser happens to use. So the snap pulls the schedule and fires
the reminders itself:

- `bin/calendo` reads the dav access token out of its own WebView — the
  session sits in localforage's IndexedDB (`localforage` / `keyvaluepairs`,
  key `session`) — and writes it to `$SNAP_USER_COMMON/session.json`, mode
  0600. It refreshes on every page load and every five minutes.
- `bin/calendo-notifier` runs from an XDG autostart entry, polls
  `listNotifications` every 15 minutes, checks for due reminders every 30
  seconds and delivers them straight over `org.freedesktop.Notifications`.
  Delivered occurrences are recorded in `notifier-state.json`, so nothing
  fires twice across restarts.

Notifications the page raises while the window is open still go through the
`show-notification` signal as before.

Two behaviours worth knowing:

- The server deletes a one-shot notification the moment it sends its own web
  push. The daemon keeps an entry that vanished from the list if it still
  owes the user that occurrence, so the race cannot swallow a reminder.
- On its very first run the daemon marks everything already due as delivered,
  so a fresh install does not replay a backlog.

Reminders only arrive while the desktop session runs — this is autostart, not
a system service. And running Calendo in a browser with web push *and* as a
snap at the same time delivers every reminder twice.

This depends on a change in Calendo and dav-js: `createAppointment` and
`createTodo` used to create the notification only if `SetupWebPushSubscription()`
succeeded, which throws where no `PushManager` exists.
