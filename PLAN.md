# nox: a tiny Linux for an old laptop

**Goal:** an old x86 laptop that boots in seconds into almost nothing, and whose only job is to run several projects side by side: the Rental Watch laptop agent, a Minecraft server, and whatever comes next. A calm **panel** on the laptop's own screen shows what's running and lets each person manage their own projects, including cloning them straight from GitHub.

## The idea in one picture

```
 ┌──────────────────────────────────────────────┐
 │  the panel: tiled windows, themes, GitHub    │  <- drawn straight to the laptop's screen
 ├──────────────────────────────────────────────┤
 │  my projects:  rental agent | minecraft | …  │  <- each in its own folder on /data
 ├──────────────────────────────────────────────┤
 │  noxd: the service runner                    │  <- one program I write: starts projects,
 │                                              │     watches them, restarts them
 ├──────────────────────────────────────────────┤
 │  init script (I write this)                  │  <- the first thing that runs at boot
 ├──────────────────────────────────────────────┤
 │  BusyBox (basic commands) + SSH + a libc     │  <- borrowed, a few MB
 ├──────────────────────────────────────────────┤
 │  Linux kernel, trimmed to this laptop        │  <- borrowed, I only choose the options
 └──────────────────────────────────────────────┘
```

Honest scope: a "distro" this small is **someone else's kernel and BusyBox plus my own glue**. The glue (init, `noxd`, the panel, build scripts) is the part I hand-code, and it's where the learning is.

## Decisions (and why)

| Question | Choice | Why |
|---|---|---|
| Build from scratch, or use Buildroot / Alpine? | **From scratch: kernel + BusyBox** | Most hand-coding, least magic. If the kernel or BusyBox build gets too painful, Buildroot is the fallback. |
| Which libc? | **musl** | Small, and Node and Java both ship musl builds (same as Alpine). Borrow musl, libstdc++ and libgcc from Alpine's tiny root filesystem rather than building them. |
| Where does the OS live? | **Kernel + OS loaded into RAM from a small boot partition. All data on a separate `/data` partition.** | Updating the OS = replace one file and reboot. Rolling back = keep the old file. A power cut can't corrupt the OS. Projects and world saves survive every update. |
| Containers? | **No. Plain processes, one Linux user per account.** | Docker/podman is big. Each project brings its own runtime (Node, Java, Python) in its own folder, so they don't clash. Revisit only if I want to run `Dockerfile`s unchanged. |
| Where does the UI show? | **On the laptop's own screen, drawn by its own program. No X, no Wayland, no browser.** | The panel's windows are its own panes (status, logs, controls), not other graphical apps, so there's nothing for a desktop system to do. It's the smallest option and the most hand-codable. The cost: the kernel needs this laptop's screen and input drivers, and the panel can't be viewed from another device (SSH covers remote admin). |
| What is the panel written in? | **Not decided yet.** Decide in phase 7. | It needs to draw to the screen, read the keyboard and mouse, and draw text with a font. Go is a candidate because it builds to one file with no libraries and can be built on Windows. |
| Is the panel part of `noxd`? | **No: a separate program that asks `noxd` what's running and tells it what to do.** | Keeps `noxd` small, and projects keep running if the panel crashes. |
| What is `noxd` written in? | **Go** (my suggestion; any language that makes one self-contained file works) | Builds to a single file with no libraries, and a Go library can clone from GitHub, so the laptop doesn't need `git` installed. |
| How do windows work? | **A tiling layout (like i3 or tmux): the screen is split into regions, each window is one region.** | Windows can never overlap and always snap into place, which is exactly the behaviour I want. Free-floating windows are much harder to keep tidy. |
| Who can use it? | **A login screen at boot. One account per person, each a real Linux user with their own projects.** | Gives every person private projects and tokens using what Linux already does. A few details are still open (see "Open questions"). |
| How do projects get onto the laptop? | **Cloned from GitHub by the panel.** A `deploy` script that copies over SSH stays as a backup. | Click-to-clone is the goal. Some projects need a setup step on the laptop (for example `npm ci`), which uses the runtime's own `npm`. |
| How does GitHub sign-in work? | **Start by pasting a fine-grained access token once** (read-only on my repos). "Sign in with GitHub" (device login) can come later. | The token route is much less code and has tight permissions. Device login needs a small GitHub app registered first. |

## The panel

### What it looks like

```
┌────────────────────────────────────────────────────────────────────────┐
│ nox                          cpu 12%   mem 1.1/4 GB   disk 31%   kyle  │
├───────────────────────┬────────────────────────────────────────────────┤
│ Projects      pinned  │ rental-agent               running · 2d 4h   x │
│ ● rental-agent        │ cpu 1%   memory 54 MB                          │
│ ● minecraft           │ [Restart] [Stop] [Settings]                    │
│ ● chat-bot    crashed │ 16:21:31 daft.ie  fetched  312 ms              │
│ ◐ blog                │ 16:22:01 daft.ie  fetched  297 ms              │
│ ○ side-project        ├────────────────────────────────────────────────┤
│                       │ minecraft                  running · 5h 12m  x │
│ On GitHub             │ memory 1.9 GB of 2 GB                          │
│ notes-app       Clone │ [Restart] [Stop] [Console]                     │
│ weather-cli     64%   │ 17:02:11 server  Steve joined the game         │
└───────────────────────┴────────────────────────────────────────────────┘
```

Extremely trimmed down: a thin bar (the laptop's health and who's logged in), then windows. Statuses and options only. The coloured mockups are in the [README](README.md).

### Windows
- The screen is always completely filled: **no overlap, no gaps.**
- Drag the line between two windows to **resize** both.
- Drag a window's title bar onto the edge of another window to **move it there**; it snaps into place. There are also layout shortcuts (one column, two columns, 2×2, big window + side strip).
- The **Projects window is always open.** It can be moved and resized but not closed.
- Clicking a project in the list opens its window. **Closing a window only hides it; it never stops the project.**
- Each account's layout is remembered, so it's the same next time they log in.

### Projects window (always there)
- Every installed project with a status dot.
- Below it, **On GitHub**: my repos, each with a **Clone** action.
- A project that has no repo on GitHub (copied over by hand) still appears in the list.

### Each project's window
- **Status** (running / starting / crashed / stopped), uptime, memory and CPU as plain numbers.
- **Live log**, scrolling as lines arrive.
- **Options:** restart, stop, and under settings: start at boot (on/off), memory limit, environment variables (with secrets hidden), **Update** (pull from GitHub, run its setup step, restart) and remove.

### Look and feel
- **Dark, terminal style:** monospace font on a blue-violet near-black, with soft, muted colours and a **blue-grey main accent** (focus, selection, links, main actions). Flat: no gradients, no glows.
- **A handful of classic terminal colour schemes** to choose from. Each theme is only a small block of about ten colours, so adding one is easy.
- **Colours always mean the same thing:** green running, yellow starting, red crashed, grey stopped.
- **Calm motion:** a running dot breathes very slightly and windows glide when resized or snapped. That's all. It switches off for anyone who prefers reduced motion.

### GitHub
1. Paste a fine-grained token once. It is stored on `/data`, readable only by that account, and never shown on screen.
2. The Projects window lists my repos. **Clone** puts one in `/data/services/<name>`.
3. `noxd` needs to know how to start it, so it looks for a tiny `nox.json` in the repo. If there isn't one, the panel asks once and remembers the answer. Example for the rental agent:
   ```json
   { "runtime": "node", "setup": "npm ci --omit=dev", "run": "node --env-file-if-exists=.env src/agent-main.js" }
   ```
4. Secrets like `AGENT_TOKEN` stay in the project's settings on `/data`, never in the repo.

### Safety
- A login screen at boot; each person has their own account and sees only their own projects and tokens.
- SSH is key-only. Nothing is served to the network except what I choose to open (for example the Minecraft port).
- If the panel ever crashes, projects keep running, and SSH plus the `svc` command still work.

### Open questions
These are deliberately undecided; none block the early phases.
- Who creates accounts, and is there an administrator?
- Can two people be logged in at once on the one screen?
- Can a project such as the Minecraft server be shared between accounts?
- Which language and drawing library will the panel use?
- How do the keyboard and mouse drive moving and resizing windows?

## Phases

Each phase has a **Done when** line, a thing I can check, so I always know where I am.

### 0. Prep (small)
- Install WSL2 and QEMU on the PC. Install Go if I'm going with Go.
- Boot any normal Linux live USB on the laptop and write down: **64-bit or 32-bit CPU**, RAM, **BIOS or UEFI**, the **screen** (size and graphics chip), whether it has an **Ethernet port**, and the **Wi-Fi chip**. Node 22 and modern Java are 64-bit only, so a 32-bit CPU would change the plan.
- Check the battery isn't swollen, and set the BIOS to "power on after power loss".

**Done when:** I have the hardware notes saved in `docs/laptop.md`.

### 1. Boot to a prompt in QEMU (medium)
- Build a small Linux kernel and a static BusyBox.
- Pack BusyBox plus a 10-line `/init` that mounts the basics and starts a shell into one file (an *initramfs*).
- Boot it in QEMU.

**Done when:** QEMU shows my own "hello from nox" and a working shell.

### 2. A real init (medium)
Extend `/init` to: set the hostname, start device handling and logging, bring up networking with DHCP, and set the clock over NTP. A wrong clock breaks HTTPS, and old laptops forget the time.

**Done when:** inside QEMU, `wget https://example.com` works.

### 3. Boot on the real laptop (medium)
- Partition a USB stick: small boot partition (bootloader, kernel + OS file) and a `/data` partition.
- Boot the laptop from it, then repeat on the laptop's own disk.
- Trim the kernel to the laptop's actual hardware. Start with the live USB's config and remove what isn't needed, **but keep this laptop's screen (framebuffer) and keyboard/mouse drivers**, because the panel will draw straight to them.

**Done when:** the laptop powers on, reaches nox with no USB attached, and `/data` is mounted read-write.

### 4. Remote access, so I can administer it from my PC (medium)
- Add **dropbear** (tiny SSH server), key login only.
- Networking on the real hardware. **Start with Ethernet** (a cheap USB-Ethernet adapter if there's no port). Wi-Fi needs a firmware file, `wpa_supplicant` and more fiddling, so it's a later bonus.
- Reserve the laptop's address in my router so I always know where it is.
- The screen dims after a few idle minutes. Closing the lid does nothing, since nothing is listening for it.

**Done when:** `ssh nox` from my PC works, and I can reboot it remotely.

### 5. `noxd`, the service runner (the first big thing I write) (medium)
Each project is a folder under `/data/services/`:

```
/data/services/rental-agent/
    nox.json   <- how to start it (and its setup step)
    env        <- its settings and secrets (locked down)
    (the project's own files)
```

`noxd` is one small program that:
- starts every service, each as the Linux user that owns it;
- restarts a service if it crashes, waiting a little longer each time;
- writes `/data/logs/<name>.log` and keeps it from growing forever;
- optionally caps each service's memory, so a hungry Minecraft can't starve the rental agent;
- answers a tiny command line: `svc list | status | start | stop | restart | logs <name>`.

`init` starts `noxd` at boot and restarts it if it ever dies. For now `noxd` has **no screen side**, only the command line.

**Done when:** two toy services run at once (say a ticking counter and a BusyBox web page). Killing one gets it restarted while the other never blinks.

### 6. First real project: the rental agent (medium)
Do this before any UI so the useful part works early.
1. Put a musl Node 22 under `/data/runtimes/node`.
2. Copy over `src/`, the few packages the agent needs (it uses `undici`) and `.env`, with a small `deploy` script on my PC that copies a folder over SSH and restarts it.
3. Its `nox.json` runs `node --env-file-if-exists=.env src/agent-main.js`. Outgoing HTTPS only, so there's nothing to open on the router. nox replaces the Windows tray app.

**Done when:** the Railway logs show `via-laptop=daft,rent` and the app stops saying the agent is offline.

### 7. Screen, input and login (medium)
Get pixels on the laptop's screen from my own program, in small steps:
1. Fill the screen with a colour, then draw a rectangle, then text in a monospace font.
2. Read the keyboard and the mouse.
3. A **login screen** that signs in a Linux account, then starts that account's panel.
4. The first, deliberately plain panel: a list of projects with their status, asking `noxd` for it. Restart and stop work from the keyboard.

**Done when:** the laptop boots to a login screen, I sign in, and I can stop and restart the rental agent from the screen.

### 8. The look: tiling windows and themes (big, and the fun part)
Build it in this order, so something works at every step:
1. **The tiling engine.** The screen is split into regions, with one divider per split and a ratio for each. Drag a divider to resize, drag a title bar to re-dock, plus the layout shortcuts. Save each account's layout as a small file.
2. **The pinned Projects window**, then a **project window** (status, numbers, live log, options).
3. **Themes.** All colours come from one set of named values, so switching a theme is one click.
4. **Motion**, last and minimal: the breathing dot and gliding windows. Respect reduced motion.

**Done when:** I can open three project windows, resize and re-dock them, log out and back in and find the same layout, and switch theme without anything looking broken.

### 9. GitHub: clone from the panel (medium)
- Settings: paste the token. The panel lists my repos in the Projects window.
- **Clone**, then read `nox.json` (or ask once), run the setup step, start the project and open its window.
- **Update** on a project window: pull, set up again, restart.

**Done when:** I clone a repo from the panel, set how it runs, start it, and see its window, without touching SSH.

### 10. Minecraft and the rest (medium)
- Java runtime under `/data/runtimes/java` and the server in `/data/services/minecraft`, with the world saved on `/data`.
- A memory cap, and a way to type console commands (a named pipe is the simple trick; later, a command box in its window).
- LAN-only at first; opening it to friends means a router port forward, which I'll decide on separately.
- Bring over any other projects through the panel.

**Done when:** the agent and Minecraft run together for 24 hours without me touching them.

### 11. Make it boring and reliable (small, ongoing)
- Backup of `/data` (including each account's layout, settings and token) to a USB drive or my PC on a schedule.
- Survives a power cut: check the filesystem on boot and come back up on its own.
- A single `make` command that rebuilds the whole OS image from scratch, so nothing lives only on the laptop.

## Repo layout

```
linux-nox/
  PLAN.md
  PRODUCT.md
  build/      scripts that fetch and build the kernel, BusyBox, dropbear and assemble the image
  config/     kernel and BusyBox settings (the .config files)
  rootfs/     files that go inside the OS: /init and /etc            <- I hand-write these
  noxd/       the service runner (Go)                                <- I hand-write this
  panel/      the on-screen UI and login screen (language not decided) <- I hand-write this
  services/   example nox.json files for my projects
  tools/      run-in-qemu, make-usb, deploy
  docs/       laptop.md, notes and the images used by the README
```

## Risks to keep in mind

- **Wi-Fi is the biggest unknown.** Ethernet first.
- **Drawing my own UI means graphics and input drivers.** The kernel needs this laptop's screen and keyboard/mouse support, and the panel needs a font and its own text drawing. Test it on the real laptop early (phase 3), not only in QEMU.
- **Old laptop RAM.** Minecraft is the hungry one (it wants a couple of GB), and it limits how many projects fit. The panel itself is small.
- **UI polish can swallow weeks.** Phase 7's plain list is already useful. Don't start motion until the tiling windows work.
- **The tiling layout is the trickiest part of the panel.** Keep it simple: splits with a ratio, never free-floating boxes.
- **Shared libraries.** Node on musl needs a few extra library files copied in. If "file not found" appears for a program that clearly exists, a missing library is why.
- **Secrets.** The agent token and each account's GitHub token live on `/data`, locked to the right user, and never in git.
- **Don't expose it to the internet casually.** Only the Minecraft port, if I choose to share it. SSH stays LAN-only.

## Later, if I want it
- A browser view of the panel, for use from another device.
- Hosting other graphical apps (this would need a Wayland compositor).
- "Sign in with GitHub" (device login) instead of pasting a token.
- Install runtimes (Node, Java) from the panel.
- Alerts to my phone when a project crashes.
- Auto-update: the laptop checks GitHub and updates a project itself.
- Run several servers with proper isolation (namespaces, or containers).
- Reach the laptop from outside the house through a tunnel such as Tailscale, instead of port forwarding.
- Wi-Fi support.
