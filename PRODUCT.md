# Product

<!-- impeccable:product-schema 1 -->

## Platform

custom Linux desktop shell, drawn straight to the laptop's own screen (not web, ios, android or adaptive)

## Stack

The UI is **its own program, drawn directly to the screen**: no X, no Wayland, no browser (confirmed by the user, 2026-10-02). Its windows are the program's own panes (project status, logs, controls), not other graphical apps.

Language and drawing library for that program are **not decided**. `PLAN.md` proposes Go for the service runner (`noxd`); the user has not confirmed it for the UI. The earlier plan of a browser-based panel is superseded.

## Users

People who own an **old x86 laptop** and want to turn it into a small always-on home server for **their own projects**. The author is the first user. Their first workloads are the Rental Watch laptop agent (repo `Kilo27/find-rentals`) and a Minecraft server.

Several people can share one laptop (confirmed 2026-10-02). There is a **login screen at boot, and each person has their own account**: a real Linux user with their own projects, who sees only those. Remote login from another computer is not part of this decision.

## Product Purpose

An extremely minimal Linux distro whose only job is to **run its users' projects side by side** and show what they are doing. It boots into a small system, starts every project, restarts the ones that crash, and shows status and a few options on the laptop's screen. It lets an old laptop run things like the Rental Watch agent, which needs an ordinary home connection that cloud hosts cannot provide, plus other projects such as a Minecraft server.

Success: the laptop runs several projects unattended, each person can see at a glance which of their projects are healthy, and they can add a project from GitHub without leaving the UI.

## Positioning

The whole desktop **is** the project dashboard. There is no general-purpose desktop, no browser, no package manager and no container runtime: a hand-built kernel + BusyBox system plus the owner's own service runner and a tiling shell that only shows projects, their status and their options.

## Operating Context

- Runs on the old laptop's own screen and keyboard/pointer. The laptop boots to a **login screen**; each person logs in to their own account. Remote access (SSH) is planned for administration, but a remote or browser view of the UI is not required.
- The OS loads into RAM from a small boot partition; everything the owner cares about lives on a separate `/data` partition that survives OS updates.
- Each project is a folder with its own runtime (Node, Java, Python…) running as a plain process under its own user. Projects are installed by cloning from GitHub.
- Most of the system is hand-coded by the author as a learning project.

## Capabilities and Constraints

- **Accounts:** one account per person, each a real Linux user with its own projects. A person sees only their own projects (confirmed 2026-10-02).
- **Windows:** every open project gets a window. Windows are resizable by the user, never overlap, and snap into place in different grid layouts.
- **Pinned window:** one window is always open: the list of all installed projects. It can be moved and resized but not closed.
- **Closing a window** only hides the view; it does not stop the project (per `PLAN.md`).
- **GitHub:** clone the owner's repos from the UI. Authentication starts with a pasted fine-grained token (per `PLAN.md`).
- **Per-project options (per `PLAN.md`):** start, stop, restart, start at boot, memory limit, environment settings, update from GitHub, remove.
- **Hardware:** old 64-bit x86 laptop; Ethernet first, Wi-Fi later. Exact hardware not yet recorded (`docs/laptop.md` is a planned Phase 0 output).
- **Terminology:** `noxd` (service runner), "the panel" (the on-screen UI), "Projects window", "nox.json" (per-project start description).
- **Undecided:** the UI language and drawing library; how the pointer and keyboard drive window moving and resizing; whether any non-built-in window (for example a terminal) is ever needed; who creates accounts and whether there is an administrator; whether two people can be logged in at once on the one screen; whether projects such as a Minecraft server can be shared between accounts; and how the GitHub token is stored per account. `PLAN.md` currently assumes a single owner and a single login, and predates this account decision.

## Brand Commitments

- Name: **nox**.
- **Dark mode with terminal themes** (the user's words), for the UI.
- **Simplicity is the point.** The UI shows statuses and options only. On 2026-10-02 the user reviewed the first mockups and asked for them to be **toned down: significantly cleaner, less gradient and less vibrant**. They liked the original colours, so the hue family stays; it is softened, not replaced. They also asked for **blue-grey as the main accent colour**. This supersedes the earlier request for a "vivid" and "lively" look.

## Evidence on Hand

- `PLAN.md` and `README.md`: the plan, and the first design mockups in `docs/img/*.svg`. The mockups use **invented example data**. They were redrawn on 2026-10-02 to show the on-device UI (no browser frame), flatter and softer, with a blue-grey main accent.
- The Rental Watch project (`Documents/rental-watch-agent`, GitHub `Kilo27/find-rentals`) is the first real workload; its laptop agent is a Node 22 program that makes outgoing HTTPS requests.
- Not on hand and not to be fabricated: any working UI, real screenshots, laptop hardware specs, or measurements.

## Product Principles

1. **Status and options only.** If something doesn't help the owner see what is running or act on it, it doesn't get screen space.
2. **One job.** The distro exists to run its users' projects. Anything that doesn't serve that job isn't in it.
3. **Understandable end to end.** The author can read and change every part, because they wrote most of it.
4. **Isolation by default.** One project crashing never affects another or the UI, and one person's projects stay invisible to the other accounts.
5. **The OS is disposable; people's data is not.** Replacing the OS never touches projects, worlds, logs or settings.
