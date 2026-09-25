# SSH to the Companion Pi over the Telemetry Radio

This guide covers switching from normal Wi-Fi SSH to SSH over the telemetry
radio link (SiK-style radio pair), for use in the field where Wi-Fi/4G isn't
available.

It works by running **PPP (Point-to-Point Protocol)** over the raw serial
link the telemetry radios provide, which turns the radio connection into a
regular IP link — so normal `ssh` works over it once it's up.

---

## How it works (read this once)

- Two telemetry radios: one plugged into the **Pi** (USB), one plugged into
  your **laptop** (USB).
- The Pi runs a `pppd` service that starts automatically on boot — **no
  Wi-Fi or prior SSH access is needed** for this to come up.
- On your laptop, you manually start a matching `pppd` client whenever you
  want to connect.
- Once both sides are up, the Pi is reachable at a fixed IP
  (`10.0.0.1`) over the radio link, and you `ssh` to it like normal.

| Device | Role     | IP address |
|--------|----------|------------|
| Pi     | server   | `10.0.0.1` |
| Laptop | client   | `10.0.0.2` |

---

## One-time setup (Pi) — do this once, needs Wi-Fi SSH access

You only need to do this section **once per Pi**. After this, the Pi
brings up the radio link on its own at every boot, without Wi-Fi.

### 1. Identify the radio's serial device

SSH into the Pi over Wi-Fi as usual, then plug in the telemetry radio and
check:

```bash
ls /dev/ttyUSB*
```

You should see `/dev/ttyUSB0`. If your setup uses a different device name,
substitute it everywhere below.

### 2. Install PPP

```bash
sudo apt update
sudo apt install ppp -y
```

### 3. Create the systemd service

```bash
sudo nano /etc/systemd/system/telem-ppp.service
```

Paste in:

```ini
[Unit]
Description=PPP over telemetry radio
After=multi-user.target

[Service]
ExecStart=/usr/sbin/pppd /dev/ttyUSB0 57600 10.0.0.1:10.0.0.2 local noauth nocrtscts nodetach
ExecStartPost=/bin/sleep 2
ExecStartPost=/sbin/ip link set ppp0 mtu 296
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> Adjust `57600` if your team's radios are configured for a different baud
> rate — it must match on both radios' own configuration and this file.

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

### 4. Enable and start it

```bash
sudo systemctl daemon-reload
sudo systemctl enable telem-ppp
sudo systemctl start telem-ppp
```

### 5. Verify

```bash
systemctl status telem-ppp
ip addr show ppp0
```

You should see `ppp0` with IP `10.0.0.1`.

**Test it:** power-cycle the Pi with no Wi-Fi and no display connected,
wait ~30 seconds after boot, and confirm the link comes up on its own
(see the laptop-side steps below to check from the other end).

This step only needs to be repeated if you re-flash the Pi's SD card.

---

## Every time you want to connect (laptop side)

No setup needed beyond this each session — just these steps:

### 1. Plug in your matching telemetry radio to the laptop

### 2. Find the device

**Windows (via WSL):**

The radio needs to be attached to WSL first, since WSL doesn't see USB
devices by default. `usbipd attach` doesn't persist across PC restarts on
its own, so set up the startup script below once — after that, this step
is automatic on every login and you can skip straight to "Start the PPP
client."

**One-time setup, install usbipd (if not already done):**

In cmd or PowerShell as Administrator:

```powershell
winget install usbipd
```

**One-time setup, auto-attach on every login:**

1. Find your radio's busid:
   ```
   usbipd list
   ```
2. Create the startup script — use whichever of these matches where
   `usbipd` works for you (cmd or PowerShell); either works the same way.

   **Option A — cmd (`.bat` file):**
   ```bat
   @echo off
   usbipd attach --wsl --busid <BUSID> --auto-attach
   ```
   Save as `attach-telem.bat` (replace `<BUSID>` with the value from
   step 1).

   **Option B — PowerShell (`.ps1` file):**
   ```powershell
   usbipd attach --wsl --busid <BUSID> --auto-attach
   ```
   Save as `attach-telem.ps1` (replace `<BUSID>` with the value from
   step 1).

   > Not sure which one has `usbipd` on its PATH? Run `where.exe usbipd`
   > in each — whichever returns a path is the one to use. If cmd works
   > and PowerShell doesn't (or vice versa), just go with the one that
   > works; no need to chase down the PATH issue.

3. Place the script in your Windows Startup folder:
   - Open it via `Win + R` → type `shell:startup` → Enter.
   - **`.bat` file:** copy the file itself directly into this folder.
   - **`.ps1` file:** PowerShell scripts don't run on double-click by
     default, so instead create a **shortcut** here pointing at:
     ```
     powershell.exe -ExecutionPolicy Bypass -File "C:\path\to\attach-telem.ps1"
     ```
     (Right-click inside the Startup folder → New → Shortcut → paste the
     line above as the location, using the actual path where you saved
     the `.ps1` file.)
4. Mark it to run as administrator (`usbipd attach` requires elevation):
   - **`.bat` file:** right-click it → Properties → if a Shortcut tab is
     present, go to Advanced → check **"Run as administrator"** → OK →
     OK. If there's no Shortcut tab, Windows will still prompt via UAC
     when it runs — that's fine.
   - **`.ps1` shortcut:** right-click the shortcut → Properties →
     Shortcut tab → Advanced → check **"Run as administrator"** → OK →
     OK.

From now on, this runs automatically on every login (you'll get one UAC
prompt to click "Yes" on each time) and the radio will already be
attached to WSL by the time you open a terminal — no manual `usbipd`
commands needed per session.

**Verify it worked**, any time:

```bash
ls /dev/ttyUSB*
```

If this comes up empty despite the startup script, check `usbipd list`
in Windows — the busid may have changed (e.g. radio plugged into a
different USB port), in which case update it in `attach-telem.bat`.

**Mac/Linux:**

```bash
ls /dev/tty.usb*      # Mac
ls /dev/ttyUSB*       # Linux
```

### 3. Start the PPP client

```bash
sudo pppd /dev/ttyUSB0 57600 10.0.0.2:10.0.0.1 local noauth nocrtscts nodetach
```

Leave this terminal running — **do not close it or press Ctrl+C in this
window**, that will tear down the link. Open a new terminal tab/window for
the next steps.

### 4. Set the MTU (new terminal, not the one running pppd)

```bash
sudo ip link set ppp0 mtu 296
```

### 5. Confirm the link is up

```bash
ping 10.0.0.1
```

You should get replies, though expect noticeably higher latency
(~150–250 ms) than Wi-Fi — this is normal over a radio link and does not
mean anything is wrong.

### 6. SSH in

```bash
ssh <username>@10.0.0.1
```

Give it up to 30–60 seconds if the prompt doesn't appear instantly — SSH's
handshake takes several round trips, and each round trip is slow on this
link.

---

## Ending a session

When you're done:

1. Close the SSH session (`exit` or `Ctrl+D`).
2. On the laptop, go to the terminal running `pppd` and press `Ctrl+C` to
   bring the client-side link down.
3. The Pi's side will automatically retry/reconnect on its own next time
   (`Restart=always` in the service), so nothing needs to be done on the Pi.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `ls /dev/ttyUSB*` shows nothing (WSL) | Radio not attached to WSL, or startup script didn't run/busid changed | Check `usbipd list` in Windows; re-run the `.bat` manually or fix the busid in it |
| `pppd: unrecognized option '/dev/ttyUSB0'` | Device doesn't exist yet | Fix the above first, then retry `pppd` |
| `LCP: timeout sending Config-Requests` then terminates | Other side's `pppd` isn't running yet | Make sure the Pi's service is active (`systemctl status telem-ppp`) and start the laptop side after |
| `ping` works but `ssh` hangs indefinitely | MTU too high, large packets dropped | Run `sudo ip link set ppp0 mtu 296` on **both** ends, in a separate terminal from the one running `pppd` |
| Link drops when you Ctrl+C | You pressed Ctrl+C in the terminal running `pppd` itself | Only Ctrl+C in the terminal running `ssh`; leave `pppd` terminals alone |
| Works fine but very slow, laggy typing | Expected — SiK-style radios are ~8–57 kbps | Normal for this link; don't `scp` large files over it, use it for lightweight shell access only |

---

## Notes for the team

- This link is meant for **field use when Wi-Fi/4G isn't available** — for
  bench testing, prefer normal Wi-Fi SSH, it's faster and simpler.
- If your team runs **two** telemetry radio pairs (one for MAVLink to the
  GCS, one dedicated to this SSH link), make sure `NETID`/frequency
  settings on each pair don't overlap.
- For anything you don't want to lose if the radio link glitches mid-task
  (e.g. running a script during a flight), run it inside `tmux` on the Pi
  so it survives an SSH disconnect:

  ```bash
  tmux new -s work
  # run your script
  # Ctrl+B then D to detach
  ```

  Reattach later with `tmux attach -t work`.
