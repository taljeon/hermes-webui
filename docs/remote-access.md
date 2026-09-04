# Remote access

How to reach a self-hosted Hermes WebUI from another machine or your phone.

## Accessing from a remote machine

The server binds to `127.0.0.1` by default (loopback only). If you are running
Hermes on a VPS or remote server, use an SSH tunnel from your local machine:

```bash
ssh -N -L <local-port>:127.0.0.1:<remote-port> <user>@<server-host>
```

Example:

```bash
ssh -N -L 8787:127.0.0.1:8787 user@your.server.com
```

Then open `http://localhost:8787` in your local browser.

`start.sh` will print this command for you automatically when it detects you
are running over SSH.

---

## Accessing on your phone with Tailscale

[Tailscale](https://tailscale.com) is a zero-config mesh VPN built on
WireGuard. Install it on your server and your phone, and they join the same
private network -- no port forwarding, no SSH tunnels, no public exposure.

The Hermes Web UI is fully responsive with a mobile-optimized layout
(hamburger sidebar, sidebar top tabs in the drawer, touch-friendly controls),
so it works well as a daily-driver agent interface from your phone.

### Preferred: Tailscale Serve

[Tailscale Serve](https://tailscale.com/docs/features/tailscale-serve) exposes
the WebUI at a tailnet-only HTTPS/MagicDNS URL while Hermes keeps listening on
`127.0.0.1`. Serve owns the network front door; continue launching Hermes
WebUI through the maintained `start.sh` or systemd path.

1. Install [Tailscale](https://tailscale.com/download) on your server and
   your iPhone/Android.
2. Before publishing anything, enable application authentication. Prefer
   configuring a strong password through Settings while reaching WebUI only
   over loopback or an SSH tunnel; Settings stores a password hash rather than
   a plaintext password. Keep the Serve cookie setting in the checkout's
   owner-only `.env`:

   ```bash
   umask 077
   touch .env
   chmod 600 .env
   ```

   ```dotenv
   HERMES_WEBUI_SECURE=1
   ```

   If Settings is unavailable, the same owner-only file can carry the password:

   ```dotenv
   HERMES_WEBUI_PASSWORD='replace-with-a-long-random-password'
   HERMES_WEBUI_SECURE=1
   ```

   `start.sh` sources `.env` as shell syntax. Keep the password single-quoted
   and do not use a literal single quote in this path; use Settings for an
   arbitrary generated password. Serve terminates HTTPS before proxying to the
   local HTTP backend, so `HERMES_WEBUI_SECURE=1` keeps the session cookie
   HTTPS-only.
3. Restart WebUI through the same lifecycle owner that already manages it. A
   process listening on `0.0.0.0:8787` also answers the loopback health probe,
   so `start.sh` would otherwise report "already running" without applying the
   new host or reloading `HERMES_WEBUI_SECURE=1`.

   For a daemon started by `ctl.sh`, keep ownership with the controller:

   ```bash
   ./ctl.sh stop
   ./ctl.sh start 8787 --host 127.0.0.1
   ```

   For a direct `start.sh` launch, stop the exact listener using the PID
   instructions that `start.sh` prints, verify it has stopped, then run:

   ```bash
   ./start.sh 8787 --host 127.0.0.1
   ```

   For a supervisor-managed service, run its matching stop command, update the
   configured launch command with `8787 --host 127.0.0.1`, then use the same
   owner to start it again:

   - systemd: `systemctl --user stop hermes-webui.service`, then
     `systemctl --user daemon-reload` and
     `systemctl --user start hermes-webui.service`.
   - launchd: `launchctl unload ~/Library/LaunchAgents/com.example.hermes-webui.plist`,
     then `launchctl load` with the same plist path.
   - supervisord: `sudo supervisorctl stop hermes-webui`, then
     `sudo supervisorctl reread`, `sudo supervisorctl update`, and
     `sudo supervisorctl start hermes-webui`.
   - runit: `sv down <service-directory>`, edit
     `<service-directory>/run` so its `exec` invokes
     `/bin/bash /path/to/hermes-webui/start.sh 8787 --host 127.0.0.1 --foreground`,
     then run `sv up <service-directory>`.
   - s6: `s6-svc -d <service-directory>`, apply the same `exec` change in
     `<service-directory>/run`, then run `s6-svc -u <service-directory>`.

   The [supervisor guide](supervisor.md) shows the corresponding launch-command
   locations. Do not launch `start.sh` directly while a supervisor owns the
   service. For example, a systemd unit uses:

   ```ini
   ExecStart=/bin/bash %h/hermes-webui/start.sh 8787 --host 127.0.0.1 --foreground
   ```

   Keep the password (unless it is already configured through Settings) and
   `HERMES_WEBUI_SECURE=1` in the checkout's `.env`; do not rely on systemd
   `Environment=` entries to override that later-loaded file. Do not continue
   until the selected lifecycle owner reports the new process running.

   For a different trusted reverse proxy that sets `X-Forwarded-Proto: https`,
   `HERMES_WEBUI_TRUST_FORWARDED_PROTO=1` is the alternative. Keep the explicit
   `HERMES_WEBUI_SECURE=1` setting for Tailscale Serve.
4. Open `http://127.0.0.1:8787` locally or through an SSH tunnel and confirm
   that an unauthenticated browser receives the login screen. Do not publish
   the Serve endpoint until this check succeeds. Use the local HTTP URL only
   for this preflight; sign in through the HTTPS Serve URL after publishing.

5. On Linux, allow the unprivileged account that runs WebUI to manage
   Tailscale. This is a one-time administrator action:

   ```bash
   sudo tailscale set --operator="$USER"
   ```

6. As that operator account, publish the loopback service in the background:

   ```bash
   tailscale serve --bg 8787
   ```

   The command prints the tailnet-only HTTPS URL to open on your phone.
   Background Serve configuration persists across device reboots and
   `tailscale down` / `tailscale up` restarts.

7. Verify the active mapping:

   ```bash
   tailscale serve status
   ```

   To remove all Serve mappings from this device, run:

   ```bash
   tailscale serve reset
   ```

Serve requires HTTPS support in the tailnet. If it is not enabled yet, the
command can print an interactive consent URL for enabling the prerequisite.
Tailnet encryption and access-control rules narrow exposure, but they do not
replace `HERMES_WEBUI_PASSWORD`.

> **Serve, not Funnel:** Serve is available only inside your tailnet.
> [Tailscale Funnel](https://tailscale.com/docs/features/tailscale-funnel) is
> public on the internet and is not the recommended path for Hermes WebUI.

For the full CLI and Linux permission contracts, see the official
[`tailscale serve` reference](https://tailscale.com/docs/reference/tailscale-cli/serve)
and [Linux operator permission guide](https://tailscale.com/docs/reference/troubleshooting/linux/linux-operator-permission).
The Simplified Chinese companion tracked in
[#7393](https://github.com/nesquena/hermes-webui/issues/7393) should mirror the
`HERMES_WEBUI_SECURE=1` Serve requirement when translated.

### Fallback: direct tailnet IP

If Serve is disabled for your tailnet or you cannot configure an operator,
complete the password setup and login-screen check above, bind WebUI to all
interfaces with an explicit launcher override, then access it through the
server's Tailscale IP. Because this fallback uses plain HTTP, change the cookie
setting before restarting WebUI while keeping the password non-empty:

```dotenv
HERMES_WEBUI_PASSWORD='replace-with-a-long-random-password'
HERMES_WEBUI_SECURE=0
```

Stop and restart through the same lifecycle owner described in step 3, changing
that owner's host argument to `0.0.0.0`. For a direct launch only, run:

```bash
./start.sh 8787 --host 0.0.0.0
```

Open `http://<server-tailscale-ip>:8787` in your phone's browser (find the IP
in the Tailscale app or with `tailscale ip -4` on the server).

Never bind Hermes WebUI to `0.0.0.0` without `HERMES_WEBUI_PASSWORD`.
`0.0.0.0` listens on every interface, not only Tailscale, so keep the host
firewall restricted as well. Traffic inside the tailnet is encrypted, but the
application still needs its own authentication boundary. You can add the page
to your home screen for an app-like experience.

### Community field report: ARM64 Android via AVF

A community report in [#2364](https://github.com/nesquena/hermes-webui/issues/2364)
documents Hermes Agent + WebUI running on a mid-range ARM64 Android phone inside
a Debian 12 VM via Android Virtualization Framework (AVF). The reported setup
used a Xiaomi Redmi Note 13 Pro 4G, 3.8 GiB RAM allocated to the VM, 8 visible
CPU cores, Chrome on Android at `localhost:8787`, and cloud-hosted inference.

This is not an official support baseline or provider/model benchmark, but it is
a useful compatibility signal for mobile ARM64 experiments: the WebUI rendered
smoothly in Chrome, ARM64 Debian worked for the agent stack, and the total local
footprint was about 1.7 GB. Practical caveats from the report: first install can
take longer when dependencies compile from source, Android browser tabs may
reload when switching apps, and disabling battery optimization for the terminal
or VM host may be needed for longer-running sessions.

> **Tip:** If using Docker, set `HERMES_WEBUI_HOST=0.0.0.0` in your
> `docker-compose.yml` environment (already the default) and set
> `HERMES_WEBUI_PASSWORD`.

---
