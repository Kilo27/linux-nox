# nox: a tiny Linux for an old laptop

**Goal:** an old x86 laptop that boots in seconds into almost nothing, and whose only job is to run several of my projects side by side: the Rental Watch laptop agent, a Minecraft server, and whatever comes next. A clean, colourful **panel** lets me see what's running and manage it, and clones my projects straight from GitHub.

## The idea in one picture

```
 ┌──────────────────────────────────────────────┐
 │  the panel: tiled windows, themes, GitHub    │  <- shown in a browser on my PC or phone
 ├──────────────────────────────────────────────┤
 │  my projects:  rental agent | minecraft | …  │  <- each in its own folder on /data
 ├──────────────────────────────────────────────┤
 │  noxd: service runner + panel server         │  <- one program I write: runs projects,
 │                                              │     restarts them, serves the panel
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
| Containers? | **No. Plain processes, one user per project.** | Docker/podman is big. Each project brings its own runtime (Node, Java, Python) in its own folder, so they don't clash. Revisit only if I want to run `Dockerfile`s unchanged. |
| Where does the UI show? | **In a web browser on my PC or phone, served by the laptop.** | The laptop needs no graphics drivers, desktop or browser, so the OS stays tiny and the UI costs almost nothing. Showing it on the laptop's own screen would need a whole graphics stack plus a browser, the opposite of minimal. It also works from the sofa. |
| What is `noxd` written in? | **Go** (my pick; any language that makes one self-contained file works) | Builds to a single file with no libraries and can be built on Windows. Its standard library already has a web server and live updates, and a Go library can clone from GitHub, so the laptop doesn't need `git` installed. |
| What is the UI built with? | **Plain HTML, CSS and JavaScript. No framework, no build step.** | Easy to hand-code and tweak, nothing to install. It's packed inside the `noxd` file, so the whole panel is one file. |
| How do windows work? | **A tiling layout (like i3 or tmux): the screen is split into regions, each window is one region.** | Windows can never overlap and always snap into place, which is exactly the behaviour I want. Free-floating windows are much harder to keep tidy. |
| How do projects get onto the laptop? | **Cloned from GitHub by the panel.** A `deploy` script that copies over SSH stays as a backup. | Click-to-clone is the goal. Some projects need a setup step on the laptop (for example `npm ci`), which uses the runtime's own `npm`. |
| How does GitHub sign-in work? | **Start by pasting a fine-grained access token once** (read-only on my repos). "Sign in with GitHub" (device login) can come later. | The token route is much less code and has tight permissions. Device login needs a small GitHub app registered first. |
| Does the UI need a password? | **Yes.** | The panel can start programs and clone code onto the box, so treat it like SSH. |

## The panel

### What it looks like

```
┌────────────────────────────────────────────────────────────────────────┐
│ nox · ● online · cpu 12% · ram 1.1/4 GB · disk 31% · up 3d   [theme ▾] │
├───────────────────────┬────────────────────────────────────────────────┤
│ PROJECTS              │ rental-agent        ● running 2d 4h   54 MB    │
│ ● rental-agent        │ [restart] [stop] [settings] [update]           │
│ ● minecraft           │ 16:22:01 job daft.ie ok                        │
│ ○ yappy-bird-bot      │ 16:22:31 job rent.ie ok                        │
│                       ├────────────────────────────────────────────────┤
│ ON GITHUB             │ minecraft           ● running 5h      1.9 GB   │
│ ⬇ ardan      [clone]  │ [restart] [stop] [settings] [update]           │
│ ⬇ TheGame    [clone]  │ 16:20:44 Steve joined the game                 │
└───────────────────────┴────────────────────────────────────────────────┘
```

Extremely trimmed down: a thin strip for the laptop's health, then windows. Nothing else.

### Windows
- The screen is always completely filled: **no overlap, no gaps.**
- Drag the line between two windows to **resize** both.
- Drag a window's title bar onto the edge of another window to **move it there**; it snaps into place. There are also layout buttons (one column, two columns, 2×2, big window + side strip).
- The **Projects window is always open.** It can be moved and resized but not closed.
- Clicking a project in the list opens its window. **Closing a window only hides the view; it never stops the project.**
- The layout is saved on the laptop, so every device shows the same one. On a phone the windows stack in a single column.

### Projects window (always there)
- Every installed project with a coloured status dot.
- Below it, **On GitHub**: my repos, searchable, each with a **Clone** button.
- A project that has no repo on GitHub (copied over by hand) still appears in the list.

### Each project's window
- **Status** (running / starting / crashed / stopped), uptime, memory and CPU as small live graphs.
- **Live log**, scrolling as lines arrive.
- **Options:** start, stop, restart, start at boot (on/off), memory limit, settings (environment variables, with secrets hidden), **Update** (pull from GitHub, run its setup step, restart) and remove.

### Look and feel
- **Dark, terminal style:** monospace font, near-black background, vivid saturated accents. A handful of classic terminal colour schemes to choose from. Each theme is only a small block of about ten colours, so adding one is easy.
- **Colours always mean the same thing:** green running, amber starting, red crashed, grey stopped.
- **Lively but not busy:** pulsing status dots, windows that glide when resized or snapped, graphs that tick live, log lines that slide in, a blinking cursor. It all switches off if the device asks for reduced motion.

### GitHub
1. Paste a fine-grained token once. It is stored on `/data`, readable only by `noxd`, and never sent to the browser.
2. The Projects window lists my repos. **Clone** puts one in `/data/services/<name>`.
3. `noxd` needs to know how to start it, so it looks for a tiny `nox.json` in the repo. If there isn't one, the panel asks once and remembers the answer. Example for the rental agent:
   ```json
   { "runtime": "node", "setup": "npm ci --omit=dev", "run": "node --env-file-if-exists=.env src/agent-main.js" }
   ```
4. Secrets like `AGENT_TOKEN` stay in the project's settings on `/data`, never in the repo.

### Safety
- One password to log in (stored hashed). LAN only; never exposed to the internet.
- Because the panel can run code, no login means no panel.
- If the network is down, SSH and the `svc` command still work.

## Phases

Each phase has a **Done when** line, a thing I can check, so I always know where I am.

### 0. Prep (small)
- Install WSL2 and QEMU on the PC. Install Go if I'm going with Go.
- Boot any normal Linux live USB on the laptop and write down: **64-bit or 32-bit CPU**, RAM, **BIOS or UEFI**, whether it has an **Ethernet port**, and the **Wi-Fi chip**. Node 22 and modern Java are 64-bit only, so a 32-bit CPU would change the plan.
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
- Trim the kernel to the laptop's actual hardware. Start with the live USB's config and remove what isn't needed.

**Done when:** the laptop powers on, reaches nox with no USB attached, and `/data` is mounted read-write.

### 4. Remote access, so I never touch its keyboard again (medium)
- Add **dropbear** (tiny SSH server), key login only.
- Networking on the real hardware. **Start with Ethernet** (a cheap USB-Ethernet adapter if there's no port). Wi-Fi needs a firmware file, `wpa_supplicant` and more fiddling, so it's a later bonus.
- Reserve the laptop's address in my router so I always know where it is.
- Screen off after a minute. Closing the lid does nothing, since nothing is listening for it.

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
- starts every service, each as its own user;
- restarts a service if it crashes, waiting a little longer each time;
- writes `/data/logs/<name>.log` and keeps it from growing forever;
- optionally caps each service's memory, so a hungry Minecraft can't starve the rental agent;
- answers a tiny command line: `svc list | status | start | stop | restart | logs <name>`.

`init` starts `noxd` at boot and restarts it if it ever dies. For now `noxd` has **no web side**, only the command line.

**Done when:** two toy services run at once (say a ticking counter and a BusyBox web page). Killing one gets it restarted while the other never blinks.

### 6. First real project: the rental agent (medium)
Do this before any UI so the useful part works early.
1. Put a musl Node 22 under `/data/runtimes/node`.
2. Copy over `src/`, the few packages the agent needs (it uses `undici`) and `.env`, with a small `deploy` script on my PC that copies a folder over SSH and restarts it.
3. Its `nox.json` runs `node --env-file-if-exists=.env src/agent-main.js`. Outgoing HTTPS only, so there's nothing to open on the router. nox replaces the Windows tray app.

**Done when:** the Railway logs show `via-laptop=daft,rent` and the app stops saying the agent is offline.

### 7. The panel's engine and a plain page (medium)
`noxd` grows a web side:
- Serves on my home network, with a **login**.
- Lists projects, starts/stops/restarts them, and streams logs and status **live** to the browser.
- Reads the laptop's health (CPU, memory, disk, temperature, uptime) for the top strip.
- The page is deliberately ugly: a list, some buttons, a log box.

**Done when:** from my PC's browser I can stop the rental agent and watch its status flip within a second, then start it again.

### 8. The look: tiling windows and themes (big, and the fun part)
Build the front-end in this order, so something works at every step:
1. **The tiling engine.** The screen is split into regions, with one divider per split and a ratio for each. Drag a divider to resize, drag a title bar to re-dock, plus the layout buttons. Save the layout as a small file on the laptop.
2. **The pinned Projects window**, then a **project window** (status, graphs, live log, options).
3. **Themes.** All colours come from one set of named values, so switching a theme is one click.
4. **Motion**, last: pulsing dots, gliding windows, live graphs. Respect reduced-motion.
5. **Phone layout:** single column.

**Done when:** I can open three project windows, resize and re-dock them, reload the page and find the same layout, and switch theme without anything looking broken.

### 9. GitHub: clone from the panel (medium)
- Settings: paste the token. `noxd` lists my repos in the Projects window.
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
- Backup of `/data` (including layout, settings and token) to a USB drive or my PC on a schedule.
- Survives a power cut: check the filesystem on boot and come back up on its own.
- A single `make` command that rebuilds the whole OS image from scratch, so nothing lives only on the laptop.

## Repo layout

```
linux-nox/
  PLAN.md
  build/      scripts that fetch and build the kernel, BusyBox, dropbear and assemble the image
  config/     kernel and BusyBox settings (the .config files)
  rootfs/     files that go inside the OS: /init and /etc            <- I hand-write these
  noxd/       the service runner + panel server (Go)                 <- I hand-write this
    web/      the panel's HTML, CSS and JS, packed into noxd
  services/   example nox.json files for my projects
  tools/      run-in-qemu, make-usb, deploy
  docs/       laptop.md and notes
```

## Risks to keep in mind

- **Wi-Fi is the biggest unknown.** Ethernet first.
- **Old laptop RAM.** Minecraft is the hungry one (it wants a couple of GB), and it limits how many projects fit. The panel itself is small.
- **UI polish can swallow weeks.** Phase 7's plain page is already useful. Don't start animations until the tiling windows work.
- **The tiling layout is the trickiest front-end part.** Keep it simple: splits with a ratio, never free-floating boxes.
- **Shared libraries.** Node on musl needs a few extra library files copied in. If "file not found" appears for a program that clearly exists, a missing library is why.
- **Secrets.** The agent token and the GitHub token live on `/data`, locked to the right user, and never in git or the browser.
- **Don't expose it to the internet casually.** Only the Minecraft port, if I choose to share it. The panel and SSH stay LAN-only.

## Later, if I want it
- Show the panel on the laptop's own screen (needs a graphics stack and a tiny browser).
- "Sign in with GitHub" (device login) instead of pasting a token.
- Install runtimes (Node, Java) from the panel.
- Alerts to my phone when a project crashes.
- Auto-update: the laptop checks GitHub and updates a project itself.
- Run several servers with proper isolation (namespaces, or containers).
- Reach the panel from outside the house through a tunnel such as Tailscale, instead of port forwarding.
- Wi-Fi support.
