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
2. Before publishing anything, enable application authentication. Create or
   edit `.env` in the WebUI checkout and set a non-empty, strong password:

   ```dotenv
   HERMES_WEBUI_PASSWORD=replace-with-a-strong-password
   HERMES_WEBUI_SECURE=1
   ```

   Alternatively, configure a password through Settings while reaching WebUI
   only over loopback or an SSH tunnel. Keep `HERMES_WEBUI_SECURE=1` in the
   service environment: Serve terminates HTTPS before proxying to the local
   HTTP backend, so this setting keeps the session cookie HTTPS-only.
3. Pin the backend address with launcher arguments, which override any
   conflicting host or port values from `.env`:

   ```bash
   ./start.sh 8787 --host 127.0.0.1
   ```

   For a systemd launch, keep the password (unless it is already configured
   through Settings) and `HERMES_WEBUI_SECURE=1` in the checkout's `.env`.
   Then adapt the unit from the [supervisor guide](supervisor.md) so its
   `ExecStart` pins host and port with arguments too:

   ```ini
   ExecStart=/bin/bash %h/hermes-webui/start.sh 8787 --host 127.0.0.1 --foreground
   ```

   Do not rely on systemd `Environment=` entries to override conflicting
   values in `.env`: `start.sh` loads that file after the service environment.
   Run `systemctl --user daemon-reload` and restart the unit after editing it.
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

### Fallback: direct tailnet IP

If Serve is disabled for your tailnet or you cannot configure an operator,
complete the password setup and login-screen check above, bind WebUI to all
interfaces with an explicit launcher override, then access it through the
server's Tailscale IP. Because this fallback uses plain HTTP, change the cookie
setting before restarting WebUI while keeping the password non-empty:

```dotenv
HERMES_WEBUI_PASSWORD=replace-with-a-strong-password
HERMES_WEBUI_SECURE=0
```

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
