---
layout: post
title: "GPU-Accelerated Remote Desktop on Linux from macOS - the Hard Way"
date: 2026-04-13 08:00:00 -0500
categories: homelab
tags: rdp gnome gpu cuda nvidia arm64 macos homelab linux
image:
  path: /assets/img/headers/gpu-accelerated-rdp.webp
  lqip: data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAASABIAAD/4QBMRXhpZgAATU0AKgAAAAgAAYdpAAQAAAABAAAAGgAAAAAAA6ABAAMAAAABAAEAAKACAAQAAAABAAAACqADAAQAAAABAAAABQAAAAD/7QA4UGhvdG9zaG9wIDMuMAA4QklNBAQAAAAAAAA4QklNBCUAAAAAABDUHYzZjwCyBOmACZjs+EJ+/8AAEQgABQAKAwEiAAIRAQMRAf/EAB8AAAEFAQEBAQEBAAAAAAAAAAABAgMEBQYHCAkKC//EALUQAAIBAwMCBAMFBQQEAAABfQECAwAEEQUSITFBBhNRYQcicRQygZGhCCNCscEVUtHwJDNicoIJChYXGBkaJSYnKCkqNDU2Nzg5OkNERUZHSElKU1RVVldYWVpjZGVmZ2hpanN0dXZ3eHl6g4SFhoeIiYqSk5SVlpeYmZqio6Slpqeoqaqys7S1tre4ubrCw8TFxsfIycrS09TV1tfY2drh4uPk5ebn6Onq8fLz9PX29/j5+v/EAB8BAAMBAQEBAQEBAQEAAAAAAAABAgMEBQYHCAkKC//EALURAAIBAgQEAwQHBQQEAAECdwABAgMRBAUhMQYSQVEHYXETIjKBCBRCkaGxwQkjM1LwFWJy0QoWJDThJfEXGBkaJicoKSo1Njc4OTpDREVGR0hJSlNUVVZXWFlaY2RlZmdoaWpzdHV2d3h5eoKDhIWGh4iJipKTlJWWl5iZmqKjpKWmp6ipqrKztLW2t7i5usLDxMXGx8jJytLT1NXW19jZ2uLj5OXm5+jp6vLz9PX29/j5+v/bAEMAFhYWFhYWJhYWJjYmJiY2STY2NjZJXElJSUlJXG9cXFxcXFxvb29vb29vb4aGhoaGhpycnJycr6+vr6+vr6+vr//bAEMBGx0dLSktTCkpTLd8Zny3t7e3t7e3t7e3t7e3t7e3t7e3t7e3t7e3t7e3t7e3t7e3t7e3t7e3t7e3t7e3t7e3t//dAAQAAf/aAAwDAQACEQMRAD8Aykl2zFCMsCQWzjPOORW8NGsiMsCT3Oa5v/l7b/e/rXcjpUmp/9k=
---

What started as "just set up RDP" turned into an all day rabbit hole about how Linux remote desktop actually works, why most of it doesn't work with NVIDIA on ARM64, and what it actually takes to get GPU-accelerated H264 encoding over RDP from an Ubuntu machine to a Mac.

---

## What I was trying to do

I have two ASUS Ascent GX10 machines running Ubuntu 24.04. They are ARM64 systems built around the NVIDIA GB10 Grace-Blackwell SoC with CUDA 13.0 and driver 580.

I just wanted to remote into them from my Mac using Microsoft's Windows App (formerly Microsoft Remote Desktop) and get a smooth, GPU-accelerated desktop.

---

## The hardware

| Component | Detail |
|-----------|--------|
| Machine | ASUS Ascent GX10 x 2 |
| SoC | NVIDIA GB10 Grace-Blackwell |
| Architecture | ARM64 (aarch64) |
| OS | Ubuntu 24.04.4 LTS |
| Kernel | 6.17.0-1014-nvidia |
| NVIDIA Driver | 580.142 |
| CUDA | 13.0 |
| GNOME | 46.0 |

These machines have real GPUs, and I wanted the remote session to actually use them for encoding the video stream instead of doing everything in software.

---

## First attempt with xrdp

xrdp is the obvious first choice on Linux. It's everywhere, it works with standard RDP clients, and it's dead simple to install.

```bash
sudo apt install xrdp
sudo systemctl enable --now xrdp
```

Connecting from the Mac worked fine. I got a desktop. But something felt off immediately.

The session was sluggish, window animations were choppy, and the whole thing just felt heavy. After some digging, I found the reason. xrdp was using software rendering.

On these machines, there's no Mesa driver for the GB10 GPU, so xrdp falls back to `swrast`, the software rasterizer. Every single frame is being rendered and encoded on the CPU while the GPU just sits there doing nothing.

```bash
$ glxinfo | grep "OpenGL renderer"
OpenGL renderer string: llvmpipe (LLVM 17.0.6, 128 bits)
```

That's `llvmpipe`. Software rendering only, no GPU.

it works, but having a machine with a real GPU and not using it for encoding video didn't seem right to me.

---

## GNOME Remote Desktop

Ubuntu 24.04 ships with GNOME Remote Desktop (gnome-remote-desktop), and it takes a totally different approach from xrdp.

Instead of creating its own X session and rendering in software, GNOME Remote Desktop hooks into the existing GNOME session and uses the system's actual GPU. On the server side, it uses FreeRDP's library for the RDP protocol.

GNOME Remote Desktop can use CUDA to encode the desktop stream as H264, specifically AVC444, which is the highest quality mode. The GPU does the encoding, and the result is a much smoother, lower-latency session.

I checked whether CUDA was available.

```bash
$ grdctl status
RDP:
    Status: enabled
```

And in the logs after starting the service.

```
[RDP] Initialization of CUDA was successful
RDP server started
```

CUDA initialized, the server started, and it was listening for connections. Nice. But actually connecting from the Mac turned out to be its own problem.

---

## The macOS Windows App problem

This is where I lost a bunch of time.

When I tried connecting from Microsoft's Windows App on macOS, it would hang at "Configuring remote PC" and just hung.

The server-side logs showed the connection arriving, TLS negotiating, and then nothing useful. Client side, it just spun.

I went through a bunch of things - switching between TLS and RDP security layers, different credential modes, toggling NLA on and off - before I found a GNOME GitLab issue that nailed it.

[GNOME Remote Desktop Issue #215](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/issues/215) documented this exact behavior. Microsoft's macOS client was sending garbled credentials during a Server Redirection PDU, which is part of GNOME Remote Desktop's auth flow.

The GNOME Remote Desktop developer's response was clear:

> "This is not something that we can work around here. This client behaviour is completely off spec. Microsoft needs to fix their client here."

The issue had been open since July 2024 at that point.

---

## VNC as a stopgap

While I was sorting out the RDP mess, I still needed some way to actually use these machines.

x11vnc got me going quick.

```bash
sudo apt install x11vnc
```

macOS has built-in VNC support through Screen Sharing (`vnc://192.168.10.66:5900`), so I had a remote desktop going in minutes. Not GPU-accelerated and the quality wasn't great, but it got me moving.

---

## FreeRDP

I ended up trying FreeRDP instead of Microsoft's client, and that's what cracked it open.

FreeRDP is open-source and available through Homebrew.

```bash
brew install freerdp
```

The `sdl-freerdp` binary uses SDL3 and Metal for rendering on macOS, so you don't even need XQuartz.

```bash
sdl-freerdp /v:192.168.10.66:3389 /u:your-username /p:PASSWORD /cert:ignore
```

Connected first try. No "Configuring remote PC" hang. No credential garbling. And the server logs showed exactly what I was hoping for.

```
[RDP] Initialization of CUDA was successful
[RDP.RDPGFX] CapsAdvertise: Accepting capability set with version RDPGFX_CAPVERSION_107,
  Client cap flags: H264 (AVC444): true, H264 (AVC420): true
```

GPU encoding the desktop as H264 AVC444, the highest quality RDP graphics mode. And `nvidia-smi` showed `gnome-remote-desktop-daemon` actively using the GPU with Compute and Graphics flags (`C+G`).

That was actual GPU-accelerated remote desktop, working.

---

## Screen lock kills everything

Before I got that working, I hit another issue that took some time to track down.

If the screen was locked, GNOME Remote Desktop would reject connections with "Session creation inhibited." Didn't matter which client, FreeRDP or Windows App, same result.

You have to disable screen lock and blanking entirely.

```bash
gsettings set org.gnome.desktop.screensaver lock-enabled false
gsettings set org.gnome.desktop.screensaver idle-activation-enabled false
gsettings set org.gnome.desktop.session idle-delay 0
```

And if the session is currently locked, unlock it.

```bash
loginctl unlock-session $(loginctl | grep your-username | awk '{print $1}')
```

For a headless-ish server that's only accessed remotely, having it lock itself and then refuse remote connections is not ideal.

---

## Dummy plugs

This one caught me off guard. GNOME Remote Desktop in mirror-primary mode needs a display to mirror.

With nothing plugged into the HDMI port, the service starts fine, but connections fail with this error.

```
Failed to record monitor: Unknown monitor
```

An HDMI dummy plug fixed it. These are just cheap little adapters that present an EDID so the system thinks a monitor is connected. Plugged one into each machine and `xrandr` showed a display.

```bash
$ DISPLAY=:0 xrandr
HDMI-0 connected primary 1920x1080+0+0
```

The dummy plugs I got report support up to 4096x2160, so I wasn't stuck at 1080p. Bumped both machines to 2560x1440.

```bash
DISPLAY=:0 xrandr --output HDMI-0 --mode 2560x1440
```

---

## Why headless mode didn't work

GNOME Remote Desktop also has a headless mode, so I tried that to avoid needing the dummy plugs.

Headless mode runs its own compositor instead of hooking into the existing GNOME session. The daemon starts, CUDA initializes, the port listens, clients connect, and you get a black screen.

The headless compositor doesn't render gnome-shell. It creates a bare compositor with nothing on it, which didn't really help for what I needed.

There's also a systemd issue. The headless daemon doesn't send `sd_notify`, so if the unit file uses `Type=notify` (which is the default), systemd waits for a signal that never comes, then kills the process after 90 seconds. You can work around it with a `Type=simple` override, but headless didn't give me a usable desktop anyway, so it was a dead end.

I also tried `extend` mode, which is supposed to add a virtual monitor to the session. Also a black screen on these machines.

Mirror-primary with a dummy plug was the only thing that actually worked.

---

## Autologin and the keyring problem

GNOME Remote Desktop runs as a user-level systemd service, so it needs a logged-in GNOME session. For a machine with no keyboard and no monitor, that means GDM autologin.

Autologin isn't something I'd recommend on a shared or exposed machine - anyone with physical access can get in. In my case these are sitting on my home network for testing, which is OK for now.  If these turn into headless servers I will disable the autologin, GNOME Remote Desktop and use ssh with `xrdp` as a fallback,

```ini
# /etc/gdm3/custom.conf
[daemon]
AutomaticLoginEnable=true
AutomaticLogin=your-username
```

The catch is that autologin through PAM doesn't create the GNOME keyring, and GNOME Remote Desktop stores its RDP credentials there. Without the keyring, the service starts but has no credentials to authenticate anyone.

I ended up writing a small systemd user service that unlocks the keyring at login.

```bash
#!/bin/bash
# /usr/local/bin/unlock-gnome-keyring
printf "sesame" | gnome-keyring-daemon --replace --unlock --components=pkcs11,secrets
```

```ini
# ~/.config/systemd/user/unlock-gnome-keyring.service
[Unit]
Description=Unlock GNOME Keyring at login
After=graphical-session.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/unlock-gnome-keyring

[Install]
WantedBy=default.target
```

With that in place, credentials survive reboots and everything comes up on its own.

---

## Persisting the resolution

xrandr settings don't survive reboots, so the 2560x1440 mode resets to 1080p every time. I added a GNOME autostart entry to set it back.

```bash
#!/bin/bash
# /usr/local/bin/set-resolution.sh
sleep 5
export DISPLAY=:0
xrandr --output HDMI-0 --mode 2560x1440
```

```ini
# ~/.config/autostart/set-resolution.desktop
[Desktop Entry]
Type=Application
Name=Set Resolution
Exec=/usr/local/bin/set-resolution.sh
X-GNOME-Autostart-Phase=Applications
X-GNOME-Autostart-enabled=true
```

The `sleep 5` is there because GNOME needs a second after login before the display is ready for xrandr.

---

## Final configuration

After all of that, here's what I landed on.

| Component | Setting |
|-----------|---------|
| RDP server | GNOME Remote Desktop 46.3 (non-headless) |
| Mode | mirror-primary |
| Port | 3389 |
| Display | HDMI dummy plug at 2560x1440 |
| Encoding | CUDA H264 AVC444 |
| GDM | Autologin enabled |
| Screen lock | Disabled |
| Keyring | Auto-unlock via systemd user service |
| Resolution | Persisted via autostart script |
| xrdp | Disabled (no GPU support on this hardware) |

Both machines are configured identically.

### Connecting from macOS

With the Windows App (v11.3.3+)

Just add a new PC with the IP address and port 3389. Enter credentials when prompted.

With FreeRDP

```bash
sdl-freerdp /v:192.168.10.66:3389 /u:your-username /p:PASSWORD /cert:ignore
```

### Verifying GPU acceleration

On the server, after connecting.

```bash
$ journalctl --user -u gnome-remote-desktop -n 10
[RDP] Initialization of CUDA was successful
[RDP.RDPGFX] CapsAdvertise: Accepting capability set with version RDPGFX_CAPVERSION_107,
  Client cap flags: H264 (AVC444): true, H264 (AVC420): true
```

```bash
$ nvidia-smi
|    0   N/A  N/A   7912    C+G   ...c/gnome-remote-desktop-daemon   392MiB |
```

The `C+G` type means the process is using CUDA Compute and Graphics, which is the GPU encoding the RDP stream.

---

## What I learned

Linux remote desktop is more fragmented than I expected.

xrdp is the easy answer, but it can't use the GPU on hardware without Mesa drivers. On NVIDIA ARM64, that means straight software rendering.

GNOME Remote Desktop can use CUDA for H264 encoding, and that's what you want on NVIDIA hardware. But there's a list of things that has to be right before it works, including autologin, keyring, a display (real or fake), screen lock disabled, and a compatible client.

Headless mode sounds like it should solve the dummy plug problem with virtual monitors and dynamic resizing. In practice it just gave me a black screen because the headless compositor doesn't run gnome-shell.

Microsoft's macOS client had a protocol bug with GNOME Remote Desktop for over a year. FreeRDP never had this issue, and it's still a good alternative if you can't get the Windows App working.

Dummy plugs are a bit janky, but they're the simplest way to give GNOME Remote Desktop something to mirror. If the plug supports decent resolutions (mine goes to 4K), it works well enough.

The setup is solid now. Survives reboots, GPU does the heavy lifting, and the desktop feels like you're sitting in front of it.

---

## Troubleshooting

Here's a quick reference for the issues that bit me.

| Symptom | Cause | Fix |
|---------|-------|-----|
| Error 0x204 | GNOME RD not running or port not listening | `systemctl --user status gnome-remote-desktop` |
| "Configuring remote PC" hang | macOS Windows App too old | Update to v11.3.3+ or use FreeRDP |
| Black screen after connect | Wrong screen-share-mode or no display | Use mirror-primary with dummy plug |
| "Session creation inhibited" | Screen is locked | Disable screen lock via gsettings |
| "Unknown monitor" error | No display connected | Plug in HDMI dummy plug |
| Service stuck in "activating" | Headless daemon + Type=notify | Do not use headless mode, or add Type=simple override |
| Sluggish xrdp session | Software rendering (llvmpipe) | Switch to GNOME Remote Desktop |
| No keyring after reboot | PAM autologin skips keyring | Add keyring unlock systemd user service |

---

🤝 Support the channel and [help keep this site ad-free](/sponsor)

⚙️ See all the hardware I recommend at <https://l.technotim.com/gear>
