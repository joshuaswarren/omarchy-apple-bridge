# omarchy-apple-bridge

Connect Apple Reminders and iMessage to an Omarchy desktop.

This package installs two tools: `vdirsyncer` and `todoman`. It ships one setup
script and two config templates. The script asks for your CalDAV server
details. Then it writes both config files.

Data flow for reminders:

```
Apple Reminders (Mac)
        |
        |  iCloudBridge on the Mac pushes changes
        v
Radicale CalDAV server (port 5232)
        |
        |  vdirsyncer pulls on a timer
        v
Local files in ~/.local/share/vdirsyncer/reminders/
        |
        v
todoman (terminal) and Planify (GUI)
```

iMessage uses a separate path. See the section "iMessage with BlueBubbles".

## Package contents

- `/usr/bin/apple-bridge-setup` — asks three questions, writes two configs.
- `/usr/share/omarchy-apple-bridge/vdirsyncer.conf.template`
- `/usr/share/omarchy-apple-bridge/todoman.conf.template`

## Prerequisites

1. An Omarchy machine with `pacman`.
2. A Radicale server on your network. Build it with option A or option B.
3. A Mac that stays on. The Mac runs iCloudBridge. iCloudBridge syncs Apple
   Reminders with Radicale. Source: <https://github.com/keithvassallomt/icloudbridge>.

## Server option A: Radicale from the Debian package (tested)

These commands were run on Debian 12 on 2026-08-22. Replace `192.168.1.50`
with your server address. Replace `yourname` with your user name.

1. Install the packages:

```bash
sudo apt install radicale apache2-utils python3-passlib
```

`python3-passlib` is required for bcrypt. The Debian `radicale` package does
not install it by itself.

2. Create the users file:

```bash
sudo htpasswd -B -c /etc/radicale/users yourname
```

The `-B` flag selects bcrypt.

3. Set `/etc/radicale/config` to:

```ini
[server]
hosts = 0.0.0.0:5232, [::]:5232

[auth]
type = htpasswd
htpasswd_filename = /etc/radicale/users
htpasswd_encryption = bcrypt

[storage]
filesystem_folder = /var/lib/radicale/collections
```

4. Fix the storage owner and start the service:

```bash
sudo chown -R radicale:radicale /var/lib/radicale
sudo systemctl enable --now radicale
```

5. Create the calendar. Run this on any machine:

```bash
curl -u yourname --request MKCALENDAR \
  --header 'Content-Type: application/xml' \
  --data '<mkcalendar xmlns="urn:ietf:params:xml:ns:caldav"><set><prop><resourcetype><collection/><calendar/></resourcetype><displayname>Reminders</displayname></prop></set></mkcalendar>' \
  http://192.168.1.50:5232/yourname/reminders/
```

`curl` asks for the password. A `201` reply confirms the creation.

## Server option B: Radicale with Docker

Use `radicale-compose.yml` from this repository. The image is the
official `kozea/radicale:stable`.

1. Create the directories:

```bash
mkdir -p radicale/config radicale/data
```

2. Save `radicale-compose.yml` from this repository as `radicale/compose.yml`.
3. Save this as `radicale/config/config`:

```ini
[server]
hosts = 0.0.0.0:5232, [::]:5232

[auth]
type = htpasswd
htpasswd_filename = /etc/radicale/users
htpasswd_encryption = bcrypt

[storage]
filesystem_folder = /var/lib/radicale/collections
```

4. Create the users file:

```bash
htpasswd -B -c radicale/config/users yourname
```

5. Start the server:

```bash
cd radicale
docker compose up -d
```

Then create the calendar with the MKCALENDAR command from option A.

## Client setup on Omarchy

1. Build and install this package:

```bash
cd packaging/omarchy-apple-bridge
makepkg -si
```

2. Run the setup script:

```bash
apple-bridge-setup
```

Answer three questions. Example URL: `http://192.168.1.50:5232/yourname/`.
The script writes `~/.config/vdirsyncer/config` with mode 600. It writes
`~/.config/todoman/config.py`.

3. Discover the calendar and run the first sync:

```bash
vdirsyncer discover reminders
vdirsyncer sync
```

4. Check the tasks:

```bash
todo list
```

5. Create a test task:

```bash
todo new -l Reminders "Bridge works"
```

## Keep the sync running

vdirsyncer does not sync by itself. A systemd user timer runs it every five
minutes.

Save `~/.config/systemd/user/vdirsyncer.service`:

```ini
[Unit]
Description=Sync reminders over CalDAV with vdirsyncer

[Service]
Type=oneshot
ExecStart=/usr/bin/vdirsyncer sync
```

Save `~/.config/systemd/user/vdirsyncer.timer`:

```ini
[Unit]
Description=Run vdirsyncer every five minutes

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
```

Enable the timer:

```bash
systemctl --user daemon-reload
systemctl --user enable --now vdirsyncer.timer
```

## GUI task app: Planify

Planify is a tasks-only GTK4 app. It is not a mail client, and it does not use
evolution-data-server. Do not install Evolution or Endeavour for this stack;
both route through GNOME PIM infrastructure that fights unattended setups.

```bash
sudo flatpak install -y flathub io.github.alainm23.planify
apple-bridge-setup   # seeds the Planify CalDAV account for you
```

`apple-bridge-setup` starts Planify once so it creates its database, then
writes the CalDAV account with `apple-bridge-planify-seed`. Two values matter:
`caldav_type` must be `generic`, and `--calendar-home-url` must be the real
collection home (`<server>/<user>/` on Radicale), because Planify assumes a
Nextcloud-style layout otherwise.

## iMessage with BlueBubbles

BlueBubbles moves iMessage off the Mac. The Mac runs the server. The Linux
machine runs the client. All code is open source.

On the Mac:

1. Install the BlueBubbles server. Download it from <https://bluebubbles.app>.
2. Sign in to Messages.app.
3. Grant Full Disk Access to the server. Open System Settings. Go to Privacy
   and Security, then Full Disk Access.
4. Open the server UI. Complete the setup wizard. The wizard sets the
   connection method and the server password.

On Omarchy:

5. Install the client:

```bash
flatpak install flathub app.bluebubbles.BlueBubbles
```

6. Open the client. Enter the server URL and the server password.

Full documentation: <https://docs.bluebubbles.app>.

## Troubleshooting

**Valid password rejected, HTTP 401.** Install `python3-passlib` on the
server. Radicale cannot read bcrypt hashes without it. Restart with
`sudo systemctl restart radicale`.

**Valid password still rejected.** Do not use `htpasswd_encryption =
autodetect`. Set `htpasswd_encryption = bcrypt`. Autodetect misread bcrypt
hashes in one deployment.

**Writes fail with HTTP 500.** The storage directory has the wrong owner. Run
`sudo chown -R radicale:radicale /var/lib/radicale`.

**Client cannot connect.** The default config binds to localhost only. Set
`hosts = 0.0.0.0:5232, [::]:5232`. Restart the service.

**vdirsyncer reports no collections.** Create the calendar first. See the
MKCALENDAR step in option A. Then run `vdirsyncer discover reminders` again.

**Password safety.** The setup script stores the password in
`~/.config/vdirsyncer/config`. It sets the file mode to 600. For more safety,
use the `password.fetch` option of vdirsyncer with a command.

## Remove

```bash
sudo pacman -R omarchy-apple-bridge
```

The files under `~/.config` stay. Delete them by hand.

## Planify: the GUI, configured without the GUI

Planify is a tasks-only GTK4 app. It is not a mail client and it does not use
evolution-data-server. Install it and let the setup script configure the
account:

```bash
sudo flatpak install -y flathub io.github.alainm23.planify
apple-bridge-setup
```

`apple-bridge-setup` starts Planify once so it creates its database, then calls
`apple-bridge-planify-seed` to write the CalDAV account. Planify syncs the lists
on the next launch. You never open its login dialog.

To seed an account by hand:

```bash
apple-bridge-planify-seed \
  --server-url http://192.168.1.50:5232/ \
  --username joshua \
  --password 'your-password' \
  --display-name 'Apple Reminders' \
  --calendar-home-url http://192.168.1.50:5232/yourname/
```

Two values matter:

- `caldav_type` must be `generic`. Planify defaults to `nextcloud`, which sends
  requests that Radicale does not answer.
- `--calendar-home-url` must be the real collection home. Planify assumes
  `<server>/calendars/<user>/`. Radicale serves `<server>/<user>/`.

### Flatpak apps do not show in the launcher

A Hyprland session can start without the flatpak export path in
`XDG_DATA_DIRS`. The app is installed but invisible. `apple-bridge-setup` links
the desktop entries into `~/.local/share/applications`, which every launcher
reads. For a permanent fix add this to `~/.config/hypr/envs.conf`:

```
env = XDG_DATA_DIRS,/usr/local/share:/usr/share:/var/lib/flatpak/exports/share:$HOME/.local/share/flatpak/exports/share
```
