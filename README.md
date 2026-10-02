<p align="center">
  <img src="docs/img/banner.svg" alt="nox: a tiny Linux for an old laptop" width="100%">
</p>

<p align="center">
  <img alt="status: planning" src="https://img.shields.io/badge/status-planning-ff9e64?style=for-the-badge&labelColor=1a1b26">
  <img alt="target: x86-64 laptop" src="https://img.shields.io/badge/target-x86--64_laptop-7aa2f7?style=for-the-badge&labelColor=1a1b26">
  <img alt="libc: musl" src="https://img.shields.io/badge/libc-musl-bb9af7?style=for-the-badge&labelColor=1a1b26">
  <img alt="noxd: Go" src="https://img.shields.io/badge/noxd-Go-7dcfff?style=for-the-badge&labelColor=1a1b26">
  <img alt="UI: vanilla JS" src="https://img.shields.io/badge/UI-vanilla_JS-e0af68?style=for-the-badge&labelColor=1a1b26">
  <img alt="containers: none" src="https://img.shields.io/badge/containers-none-f7768e?style=for-the-badge&labelColor=1a1b26">
  <img alt="hand-coded: yes" src="https://img.shields.io/badge/hand--coded-yes-9ece6a?style=for-the-badge&labelColor=1a1b26">
</p>

<h3 align="center">
  A hand-built, extremely small Linux that turns an old x86 laptop into a home server,<br>
  with a calm, terminal-style panel on the laptop's own screen.
</h3>

<br>

> **🚧 Status: planning.** Nothing is built yet. The pictures on this page are **design mockups** of what nox is meant to become, with made-up example data. **[PLAN.md](PLAN.md)** is the step-by-step route to get there.

<br>

## ✨ What is nox?

An old laptop is a perfectly good always-on computer, if it only runs what it needs to. **nox** is a Linux built from the ground up to do exactly one job: **run my projects side by side and show me how they're doing.**

No desktop, no package manager, no background services I didn't choose. It boots into a tiny system, starts every project, restarts the ones that crash, and shows a calm panel on the laptop's own screen.

**Why it exists:** the [Rental Watch](https://github.com/Kilo27/find-rentals) laptop agent has to fetch pages from an ordinary home connection, because Daft.ie and Rent.ie refuse cloud servers. A little box at home solves that, and once it's there it can also host a Minecraft server and anything else I build.

| | |
|---|---|
| 💻 **Hardware** | one old 64-bit x86 laptop, wired Ethernet to start with |
| 🧠 **Brain** | a trimmed Linux kernel + BusyBox + **`noxd`**, my own service runner |
| 🏃 **Runs** | the Rental Watch laptop agent, a Minecraft server, whatever comes next, all at once |
| 🖥️ **Panel** | on the laptop's own screen: tiling windows that never overlap, terminal themes, live status and logs |
| 👥 **Accounts** | a login screen at boot; each person has their own account and their own projects |
| 🐙 **Getting projects on** | clone them from GitHub with one click in the panel |
| 🔌 **Day to day** | work at the laptop's screen; SSH is there for admin from another computer |
| ✍️ **Hand-coded** | `/init`, `noxd`, the panel and the build scripts |

<br>

## 🧱 The stack

<p align="center">
  <img src="docs/img/architecture.svg" alt="The nox stack: panel, projects, noxd, init, BusyBox plus dropbear plus musl, Linux kernel, hardware. The top four layers are hand-written." width="100%">
</p>

A "distro" this small is **someone else's kernel and BusyBox plus my own glue**. The glue is where the learning is.

| Layer | Who | What it does |
|---|---|---|
| 🎨 **The panel** | ✍️ me | The tiling UI, drawn straight to the laptop's screen by its own program (no X, Wayland or browser). Language not decided yet |
| 📦 **Projects** | ✍️ me | Each lives in its own folder with its own runtime (Node, Java, Python…) |
| ⚙️ **`noxd`** | ✍️ me | Starts projects, restarts them, caps their memory and writes their logs. One self-contained program, planned in Go |
| 🚀 **`/init`** | ✍️ me | The first program at boot: mounts disks, brings up the network and clock, starts `noxd` |
| 🧰 **BusyBox, dropbear, musl** | 📦 borrowed | Basic commands, an SSH server and the C library, a few MB in all |
| 🐧 **Linux kernel** | 📦 borrowed | Trimmed to this laptop, keeping its screen and input. I only choose the options |

<br>

## ⏻ From power button to panel

<p align="center">
  <img src="docs/img/bootflow.svg" alt="Six boot steps: power on, firmware, bootloader, kernel, init, noxd. noxd then starts every project and the panel in parallel." width="100%">
</p>

<br>

## 💾 Where things live

<p align="center">
  <img src="docs/img/disk.svg" alt="A small boot partition holds the kernel and nox image, which is loaded into RAM. A large /data partition holds services, runtimes, logs, layout and tokens." width="100%">
</p>

The OS is **one file** that loads into RAM, and everything I care about lives on a **separate `/data` partition**. That gives three nice properties for free:

- 🔄 **Updating** the OS means replacing one file and rebooting.
- ⏪ **Rolling back** means keeping the old file.
- ⚡ **A power cut can't damage the OS**, and `/data` is checked on every boot.

<br>

## 🖥️ The panel

<p align="center">
  <img src="docs/img/panel.svg" alt="Mockup of the nox panel on the laptop's screen: a thin status bar, a pinned Projects window with a GitHub clone list, and two project windows, rental-agent and minecraft." width="100%">
</p>

<p align="center"><sub>Design mockup with example data. Everything shown is planned, not built.</sub></p>

Extremely trimmed down: a thin bar for the laptop's health (CPU, memory, disk) and who is logged in, then windows. Statuses and options only.

### 🪟 Windows that behave

<p align="center">
  <img src="docs/img/tiling.svg" alt="Four layout presets, then three steps for rearranging windows: drag a divider, drag a title bar onto an edge with a dashed preview, and the window snaps into place with no overlap." width="100%">
</p>

The screen is split into regions and every window is exactly one region. That's a **tiling layout**, like i3 or tmux, and it's why windows can never overlap and always snap into place.

- ↔️ **Drag the divider** between two windows to resize both.
- 🧲 **Drag a title bar** onto the edge of another window to dock it there.
- 🧩 **Layout buttons:** stack, side by side, 2×2, or one big window plus a side strip.
- 📌 **The Projects window is always open.** Move it, resize it, never close it.
- 🙈 **Closing a window only hides it.** The project keeps running.
- 💾 **Your layout is remembered:** each account gets the same arrangement next time it logs in.

### 🎨 Terminal themes

<p align="center">
  <img src="docs/img/themes.svg" alt="Six dark terminal colour themes with their swatches: nox default, Nord, Gruvbox, Solarized, Catppuccin and Tokyo Night." width="100%">
</p>

Dark and monospace, on a blue-violet near-black, with soft muted colour and a **blue-grey main accent** for focus, selection and links. Everything is flat: no gradients, no glows. A theme is only about ten colours, so adding my own is one small block. Motion is minimal: a running dot breathes very slightly and windows glide when resized or snapped. It all switches off for anyone who prefers reduced motion.

### 🚦 Status colours

Colours always mean the same thing, in every theme:

| | State | Meaning |
|---|---|---|
| 🟢 | **running** | Up and healthy |
| 🟡 | **starting** | Coming up, or being set up |
| 🔴 | **crashed** | Exited unexpectedly. `noxd` retries by itself |
| ⚪ | **stopped** | Switched off on purpose |

```mermaid
flowchart LR
    S(["⚪ stopped"]) -->|start| ST(["🟡 starting"])
    ST -->|came up| R(["🟢 running"])
    R -->|stop| S
    R -->|exits or crashes| C(["🔴 crashed"])
    C -->|wait a little longer each time| ST

    classDef stopped fill:#24283b,stroke:#6b7399,color:#c0caf5
    classDef starting fill:#33333d,stroke:#c3a475,color:#c3a475
    classDef running fill:#2d353d,stroke:#96b574,color:#96b574
    classDef crashed fill:#352f40,stroke:#da8494,color:#da8494
    class S stopped
    class ST starting
    class R running
    class C crashed
```

### 🧩 What's in a project window

| | |
|---|---|
| 📊 **Status** | Running / starting / crashed / stopped, uptime, memory and CPU as plain numbers |
| 📜 **Live log** | Scrolls as lines arrive |
| ▶️ **Controls** | Restart and stop up front (start when it's stopped) |
| 🔁 **Start at boot** | On or off per project, in Settings |
| 🧮 **Memory limit** | In Settings, so a hungry Minecraft can't starve the rental agent |
| ⚙️ **Settings** | Environment variables, with secrets hidden |
| ⇣ **Update** | Pull from GitHub, run the setup step, restart |
| 🗑️ **Remove** | Take the project off the laptop |

<br>

## 🐙 GitHub integration

Installing a project should be a click, not an `scp` session.

```mermaid
flowchart LR
    A(["Open the panel"]) --> B["Projects window"]
    B --> C{{"On GitHub list"}}
    C -->|Clone| D["noxd clones into /data/services/name"]
    D --> E{"nox.json found?"}
    E -->|yes| F["Run the setup step, e.g. npm ci"]
    E -->|no| G["Panel asks once and remembers"]
    G --> F
    F --> H["Start the project"]
    H --> I(["Its window opens and goes green"])

    classDef step fill:#24283b,stroke:#8fa2bc,color:#c0caf5
    classDef fork fill:#31324a,stroke:#b8a1e1,color:#b8a1e1
    classDef done fill:#2d353d,stroke:#96b574,color:#96b574
    class B,D,F,G,H step
    class C,E fork
    class A,I done
```

1. **Sign in once** by pasting a fine-grained access token (read-only on my repos). It is stored on `/data`, readable only by that account, and **never shown on screen**. "Sign in with GitHub" (device login) can come later.
2. The Projects window lists my repos under **On GitHub**, searchable. **Clone** puts one in `/data/services/<name>`. `noxd` clones by itself, so the laptop doesn't need `git` installed.
3. `noxd` needs to know how to start it, so it looks for a tiny **`nox.json`** in the repo. If there isn't one, the panel asks once and remembers. For the rental agent:

   ```json
   { "runtime": "node", "setup": "npm ci --omit=dev", "run": "node --env-file-if-exists=.env src/agent-main.js" }
   ```

4. Secrets such as `AGENT_TOKEN` live in the project's settings on `/data`, never in the repo.
5. **Update** on a project window pulls, sets up again and restarts.

<br>

## 🔐 Safety

- 🔑 **A login screen at boot.** Each person has their own account (a real Linux user) and sees only their own projects and tokens.
- 🏠 **Nothing is served to the network** except what I choose to open. The only port I might ever open is Minecraft's, if I choose to share it.
- 🗝️ **SSH is key-only.**
- 🤫 **Tokens** (GitHub, the agent) live on `/data`, locked to the right user, and are never in git.
- 🛟 **If the panel crashes,** projects keep running, and SSH plus the `svc` command line still work.

<br>

## 🧭 Roadmap

<p align="center">
  <img src="docs/img/roadmap.svg" alt="Twelve numbered phases in two rows, grouped by colour: foundation 0 to 4, run things 5 and 6, the panel 7 to 9, finish 10 and 11." width="100%">
</p>

Every phase ends with a check I can actually perform, so I always know where I am. The full detail is in **[PLAN.md](PLAN.md)**.

- [x] **Plan written**
- [ ] **0 · Prep.** WSL2 + QEMU on the PC; hardware notes from a live USB. ✅ *Done when:* `docs/laptop.md` exists
- [ ] **1 · Boot to a prompt in QEMU.** Kernel + BusyBox + a tiny `/init`. ✅ *Done when:* QEMU shows my own "hello from nox"
- [ ] **2 · A real init.** Hostname, logging, DHCP, NTP clock. ✅ *Done when:* `wget https://example.com` works in QEMU
- [ ] **3 · Boot on the real laptop.** Boot + `/data` partitions, kernel trimmed to the hardware (keeping its screen and input). ✅ *Done when:* it reaches nox with no USB attached
- [ ] **4 · Remote access.** dropbear SSH, Ethernet, fixed address. ✅ *Done when:* `ssh nox` works and I can reboot it remotely
- [ ] **5 · `noxd`, the service runner.** Start, restart on crash, logs, memory caps, `svc` command. ✅ *Done when:* two toy services run at once and one can be killed without the other noticing
- [ ] **6 · First real project: the rental agent.** Node runtime + a `deploy` script. ✅ *Done when:* Railway logs show `via-laptop=daft,rent`
- [ ] **7 · Screen, input and login.** Draw to the screen, read the keyboard and mouse, a login screen, a plain project list. ✅ *Done when:* I boot to a login screen, sign in and restart the rental agent from the screen
- [ ] **8 · The look.** Tiling engine, project windows, themes, minimal motion. ✅ *Done when:* three windows resize, re-dock and survive logging out and back in
- [ ] **9 · GitHub clone.** Token, repo list, clone, `nox.json`, update. ✅ *Done when:* I clone, configure and start a repo without touching SSH
- [ ] **10 · Minecraft and the rest.** Java runtime, memory cap, console. ✅ *Done when:* agent and Minecraft run together for 24 hours untouched
- [ ] **11 · Make it boring.** Backups, power-cut recovery, one-command rebuild. ✅ *Done when:* I can rebuild the whole OS image with one `make`

> 🏁 **Milestone:** the rental agent runs on the laptop after phase 6, before the panel exists.

<br>

## 🧰 Decisions at a glance

| Question | Choice | Why |
|---|---|---|
| From scratch, Buildroot or Alpine? | **From scratch: kernel + BusyBox** | The most hand-coding and the least magic. Buildroot is the fallback |
| libc | **musl** | Small, and Node and Java both ship musl builds |
| Where does the OS live? | **In RAM, loaded from a small boot partition**, data on `/data` | Easy updates and rollbacks, safe from power cuts |
| Containers? | **No.** Plain processes, one user per project | Docker is big. Each project brings its own runtime |
| Where does the UI show? | **On the laptop's own screen, drawn by its own program** | No X, Wayland or browser: the smallest option, and the most hand-codable. The windows are the panel's own panes |
| Panel and `noxd` | **Separate programs** | Projects keep running if the panel crashes |
| `noxd` language | **Go** (suggested) | One static file, builds on Windows, a Go library can clone from GitHub |
| Panel language | **Not decided yet** | Needs screen drawing, keyboard and mouse input, and text rendering. Go is a candidate |
| Accounts | **A login screen at boot, one account per person** | Each is a real Linux user with private projects and tokens |
| Window behaviour | **Tiling layout** | Windows can't overlap and always snap |
| GitHub sign-in | **Fine-grained token first** | Much less code and tight permissions |

<br>

## ⚠️ Risks to keep in mind

- 📶 **Wi-Fi is the biggest unknown.** It needs firmware files and `wpa_supplicant`. Ethernet comes first.
- 🧠 **Old laptop RAM.** Minecraft is the hungry one and it limits how many projects fit. The panel itself is small.
- 🖼️ **Drawing my own UI needs graphics and input drivers.** The kernel must keep this laptop's screen and keyboard/mouse support, and the panel needs a font and its own text drawing. Test on the real laptop early, not only in QEMU.
- 🎭 **UI polish can swallow weeks.** Phase 7's plain list is already useful. No motion until the tiling windows work.
- 🧱 **The tiling layout is the trickiest part of the panel.** Keep it simple: splits with a ratio, never free-floating boxes.
- 📚 **Shared libraries.** Node on musl needs a few extra library files copied in. "File not found" for a program that clearly exists usually means a missing library.
- 🔋 **Always-on laptop.** Check the battery isn't swollen before leaving it plugged in.

<br>

## 📁 Planned repo layout

```
linux-nox/
├── PLAN.md
├── PRODUCT.md
├── README.md
├── build/      scripts that fetch and build the kernel, BusyBox, dropbear and assemble the image
├── config/     kernel and BusyBox settings (the .config files)
├── rootfs/     files that go inside the OS: /init and /etc            <- hand-written
├── noxd/       the service runner (Go)                                <- hand-written
├── panel/      the on-screen UI and login screen                      <- hand-written
├── services/   example nox.json files for my projects
├── tools/      run-in-qemu, make-usb, deploy
└── docs/       laptop.md, notes and the images used by this README
```

<br>

## 🔮 Later, if I want it

- 🌐 A browser view of the panel, for use from another device
- 🪟 Hosting other graphical apps (this would need a Wayland compositor)
- 🔓 "Sign in with GitHub" (device login) instead of pasting a token
- 📥 Install runtimes (Node, Java) from the panel
- 🔔 Alerts to my phone when a project crashes
- 🤖 Auto-update: the laptop checks GitHub and updates a project itself
- 🧱 Proper isolation between servers (namespaces, or containers)
- 🌍 Reach the laptop from outside the house through a tunnel such as Tailscale
- 📡 Wi-Fi support

<br>

## 🚀 Where to start

Phase 0, in about an evening:

1. Install **WSL2** and **QEMU** on the PC (and Go, if `noxd` is going to be Go).
2. Boot any normal Linux **live USB** on the laptop and write down: 64-bit or 32-bit CPU, RAM, BIOS or UEFI, the screen and its graphics chip, Ethernet port, Wi-Fi chip. *Node 22 and modern Java are 64-bit only.*
3. Check the battery, and set the BIOS to **power on after power loss**.
4. Save the notes in `docs/laptop.md`.

<br>

## 🙏 Standing on shoulders

nox borrows the [Linux kernel](https://kernel.org), [BusyBox](https://busybox.net), [musl](https://musl.libc.org), [dropbear](https://matt.ucc.asn.au/dropbear/dropbear.html) and [Go](https://go.dev), and runs projects on [Node.js](https://nodejs.org) and Java. The panel's terminal themes are inspired by the classic colour schemes Tokyo Night, Gruvbox, Nord, Catppuccin and Solarized.

<p align="center">
  <br>
  <sub>The pictures in <code>docs/img</code> are plain SVG: edit them in any text editor.</sub>
</p>
