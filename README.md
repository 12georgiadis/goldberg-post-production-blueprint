# Goldberg Post-Production Blueprint

**An architectural reference for a 5-person AI-assisted documentary post-production, April 2026 — End of 2027.**

By [Ismaël Joffroy Chandoutis](https://ismaeljoffroychandoutis.com/) — filmmaker, Paris.

---

## What this is

This repository documents the complete post-production architecture for *The Goldberg Variations*, a feature documentary in production at [Films Grand Huit](https://filmsgrandhuit.com) with support from Villa Albertine. It is published as a reference blueprint for other independent documentary teams working at the intersection of traditional cinema pipelines and AI-assisted workflows.

The goal is not to prescribe. It is to show one concrete, working, tight-budget architecture for a five-person team shipping a film to three different delivery standards (festival DCP, streaming, Netflix HDR IMF) over twenty months, using Final Cut Pro as the creative hub, DaVinci Resolve as a specialist color tool, and Claude Code as the orchestration layer across the whole pipeline.

If you are writing a CNC dossier, an IFCIC application, a Procirep budget, or just trying to figure out how to put real numbers against an AI-integrated documentary pipeline in 2026, this is meant to help.

---

## Project context

*The Goldberg Variations* is a feature documentary about Joshua Ryne Goldberg, an autistic young man from Florida whose online life generated dozens of personas across 2010-2016 ideological forums, and the question of what a documentary can see of a person who existed mostly in text. The film uses archival footage (internet captures, forum screenshots, streams from 2013-2015), reconstructions shot in April 2026 on Blackmagic Pocket 6K, direct material with Joshua post-release, and AI-generated reconstructions of spaces and personas.

**Key production facts:**

| Fact | Value |
|---|---|
| Director | Ismaël Joffroy Chandoutis |
| Production | Films Grand Huit (Paris) |
| Post-production start | April 2026 |
| Picture lock target | End of 2027 |
| Team size | 5 people (editing) + 3 VFX |
| Edit platform | Final Cut Pro 12.2 |
| Color platform | DaVinci Resolve 20 Studio (colorist, Windows) |
| Audio mix platform | Pro Tools (external mixer, France) |
| Delivery targets | Festival DCP, streaming, Netflix HDR IMF |
| Working color space | ACES 1.3 ACEScct |

The blueprint below reflects the architecture as of April 15, 2026, after three weeks of initial work on the SpliceKit MCP integration, PR #51 to upstream SpliceKit, and extended research on the FCP ↔ Resolve ↔ Pro Tools roundtrip problem in a modern AI-assisted pipeline.

---

## Why these choices

Before the diagrams, a word on philosophy. Three decisions shape the whole architecture:

**1. Final Cut Pro as the creative hub, not as a solo NLE.** FCP 12.2 with SpliceKit MCP is, in April 2026, the most programmable NLE in the world. A dylib injected into FCP's process exposes 78,000+ ObjC classes via JSON-RPC. Claude Code can call 200+ tools directly inside the running FCP process: `blade_at_times()`, `get_transcript()`, `detect_scene_changes()`, `apply_effect()`, `export_xml()`. No AppleScript, no UI automation, no roundtrip delay. See [the accompanying essay on this shift](https://github.com/12georgiadis/deep-research/tree/main/10-the-timeline-becomes-readable). FCP's historical weakness (single-user, no Windows) is offset here by the agentic programmability, which redraws the collaboration axis in a way traditional benchmarks do not capture.

**2. DaVinci Resolve as a specialist tool, not as an alternative NLE.** Resolve is the industry's best color tool and the only viable IMF packaging tool for Netflix delivery. It is not used as a general-purpose editor in this pipeline. The team works in FCP, exports FCPXML to Resolve for final grade and IMF wrap, and comes back to FCP only for picture finishing. The colorist operates on Windows with their own Resolve Studio license and their own HDR reference monitor.

**3. Claude Code as the orchestration layer, not as a replacement for the editor.** Throughout the pipeline, Claude Code (Anthropic CLI + MCP servers) orchestrates repetitive work, indexes rushes, generates AI footage, maintains documentation, and coordinates between tools. It is not the editor. The editor makes the decisions. The agent is a collaborator that reads 200 hours of material faster than any human assistant ever could, but it never commits a cut. Every cut is human.

See also [the 2026 editing map](https://github.com/12georgiadis/deep-research/tree/main/11-the-2026-editing-map) for the full comparative framing that led to these choices.

---

## High-level architecture diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         GOLDBERG POST-PRODUCTION                    │
│                           April 2026 → End 2027                     │
└─────────────────────────────────────────────────────────────────────┘

[ PRODUCTION CAPTURE ]
   Blackmagic Pocket 6K / Cinema Camera 6K → BRAW 12:1
             │ MHL + xxHash ingest + R2 Cloudflare backup
             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  NAS UGreen DXP 8800 Plus (8-bay RAID 6, dual 10GbE, NVMe cache)    │
│  ├── /01_rushes_natives/   → BRAW sources                           │
│  ├── /02_rushes_proxies/   → ProRes 422 LT 1080p (auto-generated)   │
│  ├── /03_fcp_libraries/    → 5 divided libraries (see below)        │
│  ├── /04_assets/           → LUTs, fonts, Motion templates          │
│  ├── /05_vfx_outputs/      → Blender / ComfyUI renders              │
│  ├── /06_audio/            → music, sound design, Logic sessions    │
│  └── /07_exports/          → masters + deliverables                 │
└─────────────────────────────────────────────────────────────────────┘
             │                       │                      │
             ▼                       ▼                      ▼
[ EDITING TEAM: 5 MACS ]    [ VFX TEAM: NOMAD ]    [ COLORIST: WINDOWS ]
   │                               │                      │
   │ Mac Mini M4 /                 │ Windows Nomad        │ Resolve Studio 20
   │ MacBook Air M3 /              │ RTX 5090 × 4 rig     │ HDR reference
   │ future Mac Studio M5 Ultra    │ 256GB+ RAM           │ monitor ✓
   │                               │                      │
   │ FCP 12.2 + SpliceKit MCP      │ ComfyUI + Flux.2     │ ACES 1.3
   │ + Color Finale 2 Pro          │ Wan 2.2, LTX-2       │ + ColorTrace
   │ + BRAW Toolbox                │ Blender 4.4          │
   │ + Motion + Logic Pro          │ + Pallaidium         │
   │ + Compressor + Pixelmator     │ Unity 6 (backup)     │
   │                               │                      │
   │ Claude Code orchestration :   │ output ProRes 4444   │ FCPXML 1.10
   │ - SpliceKit MCP (FCP direct)  │ / EXR to NAS         │ round trip
   │ - comfyui-mcp                 │                      │
   │ - framelink-figma MCP         │                      │
   │                               │                      │
   └──────── PostLab alternative ──┘                      │
      (5 divided libraries, free coordination)            │
                                                           ▼
[ SOUND ]                                           [ DELIVERY ]
   FCP → Logic Pro (intra-Apple roundtrip)              │
   FCP → X2Pro → Pro Tools (mixer France)               ├── DCP festival
                                                         │   (Compressor, P3-D65 SDR)
                                                         │
                                                         ├── Streaming H.265
                                                         │   (Compressor, Rec.709)
                                                         │
                                                         └── IMF Netflix HDR
                                                             (Resolve Studio,
                                                              Dolby Vision / HDR10)
```

---

## Hardware infrastructure

### Shared storage (central to the whole team)

**UGreen DXP 8800 Plus**, 8 bays, dual 10GbE, NVMe SSD cache slots populated, Intel x86 (supports TrueNAS flash if needed later). RAID 6 across 8 HDDs for ~60-120 TB usable depending on drive size. This is overkill for a 5-person documentary team, which is exactly right: the NAS is not the bottleneck, and there is headroom for the full 20-month project including all iterations, VFX renders, and master archives.

**Why dual 10GbE matters**: with LACP bonding or multi-path routing, the NAS can sustain 20 Gbps aggregate throughput. In practice, five editors on ProRes 422 LT 1080p proxies consume roughly 5% of that. The headroom absorbs VFX renders pushing EXR sequences from Nomad and master renders from Resolve simultaneously.

**Why NVMe cache matters**: read cache on M.2 NVMe means hot files (the current proxies, the current project metadata, the FCPXML exports) live at SSD speed while cold files (archived rushes, past versions) live on spinning disk. The team feels SSD latency on the everyday workflow without paying for 60 TB of NVMe.

### Editing workstations

| Role | Machine | RAM | Notes |
|---|---|---|---|
| Lead editor (Isma) | Mac Mini M4 → Mac Studio M5 Ultra Q4 2026 | 16GB → 64/128GB | Proxy edit now, native edit when M5 Ultra ships |
| Lead editor mobile | MacBook Air M3 | 16GB | Proxies only, travel and review |
| Editor 2 (rushes/ingest) | Own Mac | 16-32GB | Proxies from NAS |
| Editor 3 (archives) | Own Mac | 16-32GB | Proxies from NAS |
| Editor 4 (interviews) | Own Mac | 16-32GB | Proxies from NAS |
| Editor 5 (reconstructions) | Own Mac | 16-32GB | Proxies from NAS |

**On the proxy question**: all five editors work on **ProRes 422 LT 1080p proxies** generated automatically at ingest. Not ProRes Proxy (too compressed for some fine cut work), not ProRes 422 HQ (overkill, slower). LT at 1080p is the sweet spot for editing 6K BRAW on Apple Silicon: decode is trivial, scrub is instant, the viewer shows at native resolution without downscaling cost, and quality is good enough for creative decisions. Native 6K BRAW is reserved for final export passes and specific QC checks, handled via FCP's Optimized/Original toggle.

### Real-time HDR preview (optional)

**iPad Pro M4** with Tandem OLED, 1600 nits peak HDR, 1000 nits sustained full-screen, P3 wide gamut, Dolby Vision and HDR10 native. Used as a second screen HDR preview during edit via AirPlay from the Mac. Not a certified mastering reference (that lives with the colorist), but sufficient to catch obvious HDR problems during the cut. ~€1800 if not already owned.

### VFX / AI rig (separate from edit)

**Nomad Windows workstation with RTX 5090 × 4 rig**, 256GB+ RAM, large NVMe arrays. Hosts:
- ComfyUI with Flux.2, Wan 2.2, LTX-2, custom LoRA trained on Joshua's visual footprint
- Blender 4.4 with Pallaidium integration for in-VSE AI generation
- Unity 6 LTS as backup for real-time / interactive experiments
- Optional: reference implementation of the [Lightricks HDR latent alignment paper](https://arxiv.org/abs/2604.11788) as the open-source HDR AI video generation becomes available

**Why Windows + 4× 5090**: ComfyUI batch generations and Blender Cycles renders are CUDA-heavy and want as many GPUs as possible. Apple Silicon is strong for editing but CUDA ecosystem for generative AI in 2026 is still Nvidia-first. The Nomad rig is the brute force layer. It does not run Final Cut Pro.

### Network and backup

- **10GbE switch** with 8-16 ports (MikroTik CRS309 or Netgear XS508M class)
- **Cat 6a shielded** for runs under 10 meters, fiber SFP+ for longer
- **Tailscale mesh** for secure remote access to all machines, including the Nomad rig from the Mac team
- **Backup strategy (3-2-1)**:
  - Copy 1: NAS (working storage)
  - Copy 2: secondary local drive (shuttle NVMe or small secondary NAS)
  - Copy 3: R2 Cloudflare (off-site, already in place)

---

## Software stack

### Apple ProApps core (editing team, each Mac)

| Tool | Version | Role | Cost each |
|---|---|---|---|
| Final Cut Pro | 12.2 | Master NLE | €299.99 |
| Motion | 5.8 | Titles, motion graphics, FCP template generation | €49.99 |
| Logic Pro | 11 | Music composition, sound design preliminary | €229.99 |
| Compressor | 4.8 | DCP, H.264/H.265, batch encode | €49.99 |
| Pixelmator Pro | Latest | Still graphics, mattes, posters, logos | €49.99 |

### FCP ecosystem plugins (editing team, at least one license each)

| Plugin | Publisher | Role | Approx cost |
|---|---|---|---|
| Color Finale 2 Pro | Color Trix | Primary grade inside FCP, ASC CDL export | $99-149 |
| BRAW Toolbox | Color Trix / Boris FX | Native Blackmagic RAW import to FCP | $79 |
| Gyroflow Toolbox | fcp.cafe | Gyroscopic stabilization | free |
| Marker Toolbox | LateNite Films | Review markers from Vimeo/Frame.io/Wipster | bundle |
| Fast Collections | LateNite Films | Smart Collections batch generation | bundle |
| CommandPost | free / community | Hardware panels, batch tagging, Lua automation | free |
| LUTx | FxFactory | LUT management | free / paid |

### Specialist tools (not on every machine)

| Tool | License | Who owns it |
|---|---|---|
| DaVinci Resolve 20 Studio | Perpetual, ~€319 | Colorist, Windows, already owned |
| Pro Tools Studio | Subscription or perpetual | External mixer in France |
| X2Pro Audio Convert | Perpetual, ~$199 | Lead editor, already owned |
| PostLab Team | $29-49/user/month | **Not used in this architecture.** See divided libraries approach below. |

### Claude Code + MCP layer

| Component | Role |
|---|---|
| Claude Code CLI | Agentic orchestration across all tools |
| SpliceKit MCP | Direct in-process FCP control (200+ tools) |
| comfyui-mcp | ComfyUI workflow orchestration from Claude |
| framelink-figma MCP | Figma as typography / graphic design source |
| fcp-xml MCP (DareDev256) | FCPXML-level batch operations (complement to SpliceKit) |
| custom Python automation | NAS cleanup, MHL verification, rush indexing |

### Open-source / free tools used throughout

- Blender 4.4 (3D, VSE, Compositor)
- ComfyUI (AI generation)
- Pallaidium (Blender plugin for AI generation)
- ffmpeg (transcode, analysis, packaging)
- MHL tools (ASC MHL, Pomfort MHL for integrity)
- Tailscale (mesh networking)

---

## Team structure

**5 people on editing + 3 VFX + external colorist + external mixer = 9 people total, but only 5 simultaneously inside the FCP project tree.**

### Editing team (5, inside FCP)

| Role | Owner of | Tools | Where |
|---|---|---|---|
| **Lead editor (Ismaël)** | `00_MASTER.fcplibrary` | FCP + Motion + Logic + Compressor + Color Finale 2 + Claude Code orchestration | Paris, Mac Mini M4 / future Mac Studio M5 Ultra |
| **Editor 2 — rushes and ingest** | `01_rushes_inventory.fcplibrary` | FCP + BRAW Toolbox + CommandPost | Own Mac |
| **Editor 3 — internet archives** | `02_archives.fcplibrary` | FCP + custom scripts for archive normalization | Own Mac |
| **Editor 4 — interviews** | `03_interviews.fcplibrary` | FCP + ScriptStar + native FCP 12 transcription | Own Mac |
| **Editor 5 — reconstructions** | `04_reconstructions.fcplibrary` | FCP + BRAW Toolbox + Color Finale 2 | Own Mac |

### VFX / AI team (3, on Nomad Windows rig)

| Role | Tools | Output |
|---|---|---|
| **VFX lead** | ComfyUI + Blender + custom LoRA pipeline | ProRes 4444 clips to NAS for FCP integration |
| **VFX 2** | ComfyUI + Flux.2 / Wan 2.2 / LTX-2 generations | Clip libraries for FCP |
| **VFX 3** | Blender + Pallaidium + (optional Unity 6) | 3D reconstructions, previz, interactive experiments |

### External contributors

| Role | Platform | Communication |
|---|---|---|
| **Colorist** | Windows + Resolve Studio 20 + HDR reference monitor | FCPXML 1.10 round trip from/to FCP |
| **Sound mixer** | Pro Tools (France) | AAF via X2Pro, printmaster return |
| **Production (Films Grand Huit)** | reviews, feedback, budget oversight | Discord + Notion + shared exports |

---

## Pipeline workflow protocols

### 1. Capture and ingest

1. Camera cards arrive from shoot (Hadrien DIT)
2. DIT copies to two separate drives with `ascmhl create -h xxh64` for integrity
3. Primary copy pushed to NAS `/01_rushes_natives/`
4. Editor 2 runs FCP import with "Create optimized media" and "Create proxy media" enabled (ProRes 422 LT 1080p)
5. Proxies populate NAS `/02_rushes_proxies/` automatically
6. A nightly cron script rsyncs the day's natives + MHL to R2 Cloudflare for off-site backup

### 2. Editing (divided libraries approach, no PostLab)

The five editors work on **strictly non-overlapping libraries**. Each library lives on the NAS as the canonical copy, and each editor shuttles a local working copy to their SSD for the session, then pushes back to the NAS at the end.

**Coordination protocol (free)**:

A single text file `LOCKS.md` in the NAS root is updated via a small CLI tool (`goldberg lock`, `goldberg unlock`, `goldberg status`), written in Python, backed by a JSON file on the NAS. Editors declare their working hours before starting their session:

```
# Goldberg — Active locks
## 2026-05-03

- 00_MASTER       : Isma    (working 09:00 - 18:00)
- 01_rushes       : Alex    (working 10:00 - 17:00)
- 02_archives     : Marie   (working 08:00 - 15:00)
- 03_interviews   : FREE
- 04_reconstructions : Lou  (working 14:00 - 22:00)
```

**Merge protocol**:

Editor 2-5 deliver finished work to the master library via FCPXML export (`File → Export XML → Current Project`). Ismaël imports the FCPXML into `00_MASTER.fcplibrary` on a daily or weekly cadence. The master editor decides what enters the canonical cut.

This is how most pro documentary teams actually work, and it is simpler, cheaper, and more conflict-free than tools like PostLab for a documentary context (as opposed to a drama series where multiple editors cut the same timeline simultaneously).

### 3. VFX and AI generation

VFX team works on the Nomad Windows rig, disconnected from the FCP tree. Their deliverables flow one direction: NAS → FCP integration.

Typical cycle:

1. Lead editor identifies a shot that needs VFX or AI generation
2. Claude Code sends a brief to the VFX team via Discord or custom MCP bridge
3. VFX team runs ComfyUI / Blender / Pallaidium workflow on Nomad
4. Output renders as ProRes 4444 or EXR sequences to `/05_vfx_outputs/<shot_id>/`
5. A Claude Code script on the Mac side detects new files and imports them into the relevant FCP library via SpliceKit `import_fcpxml()` or direct media add
6. Editor places the new clip in the timeline

For AI generation specifically, the ComfyUI workflows are committed to a shared workflow library (Pinecone index for semantic retrieval, or simple Git repo) so the team can reuse proven generation prompts across shots.

### 4. Color grade (FCP → Resolve → back to FCP)

**Round trip protocol**:

1. Ismaël exports a **partial FCPXML 1.10** containing only the sequences or clips ready for grade
2. He consolidates media associated with that partial (so the colorist has a clean package)
3. He generates an ASC MHL for the package
4. Delivery to colorist via NAS, cloud drop, or shuttle drive, depending on schedule
5. Colorist imports into Resolve Studio (Windows), conforms, grades in ACES 1.3 ACEScct working space
6. Colorist renders graded clips as individual ProRes 4444 files with the same source filename plus `_GRADED` suffix
7. Graded ProRes files returned to NAS
8. Ismaël in FCP swaps the original clips for graded clips using SpliceKit batch operations
9. Final master assembly continues in FCP

**Iteration case**: if the cut changes after clips have been graded, Ismaël re-exports an updated partial FCPXML. The colorist uses Resolve's **ColorTrace** to reapply the existing grades to the new cut, minimizing re-grade work. Only new or trimmed clips need fresh grade attention.

### 5. Sound (FCP → Logic Pro → FCP → Pro Tools → FCP)

**Preliminary in-house**:

1. Ismaël builds sound preliminary inside FCP with audio roles assigned (Dialogue.Narration, Dialogue.Sync, Music.Score, Effects.Ambience, Effects.Hard)
2. For music composition and more elaborate sound design, `File → Send Audio to Logic Pro`. Logic opens with all audio on separate tracks.
3. Composition / design work happens in Logic
4. Bounce stems back to FCP via AIFF / WAV export, placed on the timeline
5. Preliminary mix is done inside FCP

**Final mix**:

1. At picture lock, Ismaël exports **AAF via X2Pro Audio Convert**. Audio roles become Pro Tools tracks automatically.
2. Mixer in France opens the AAF in Pro Tools, imports a reference QuickTime ProRes 422 HQ with burn-in (timecode, clip name, slate)
3. Mixer produces the final mix, delivers stems (dialogue, music, effects) and a printmaster
4. Ismaël imports the printmaster into FCP on a new audio track, mutes the original preliminary audio
5. Final master image + sound bounces from FCP

### 6. Delivery (three targets)

| Target | Format | Tool | Color | Audio |
|---|---|---|---|---|
| **Festival DCP** | JPEG2000 in MXF, SMPTE DCP | Compressor 4.8 | P3-D65 SDR | 5.1 or stereo from printmaster |
| **Streaming** | H.264 or H.265 MP4 | Compressor 4.8 | Rec.709 SDR | stereo downmix |
| **Netflix IMF** | IMF 2067-21, HDR10 or Dolby Vision | **Resolve Studio** (exported from Resolve, not FCP) | Rec.2020 PQ HDR | Full stem package |

The Netflix IMF delivery is the one case where the file leaves the FCP ecosystem permanently: the master ProRes 4444 XQ is exported from FCP to Resolve one last time for IMF packaging and HDR trim, because FCP does not export IMF natively. This is accepted as a one-way operation at the end of the chain.

---

## ACES working space

The entire pipeline works in **ACES 1.3 ACEScct** from ingest to delivery, which unlocks three benefits:

1. **Coherent color pipeline** between FCP (ACES Wide Gamut HDR color space), Blender (OCIO ACES config), ComfyUI (with appropriate HDR nodes), and Resolve (ACEScct timeline).
2. **Single master, multiple deliveries**: the ACES master can be trimmed to P3-D65 SDR for festival DCP, Rec.709 for streaming, and HDR10 or Dolby Vision for Netflix, without re-grading each target.
3. **Future-proof**: if the film is later re-delivered in another format (Rec.2020 PQ, HDR Vivid, any future standard), the ACES master is still usable.

**Monitoring considerations**: the ASUS ProArt monitors used by the editing team are not HDR reference monitors. They are SDR Rec.709 / P3-D65 at ~400 nits. The editing team sees the SDR ODT only. The colorist has a proper HDR reference monitor (Flanders Scientific, Sony BVM or equivalent) for the HDR trim. The iPad Pro M4 serves as an informal HDR preview for review moments during the edit.

---

## Budget

### Immediate software (one-time, already acquired as of April 15, 2026)

| Item | Cost | Status |
|---|---|---|
| Color Finale 2 Pro (1 license) | ~$150 | ✓ acquired |
| BRAW Toolbox (1 license) | ~$80 | ✓ acquired |
| Compressor 4.8 | ~€50 | ✓ acquired |
| Motion 5.8 | ~€50 | ✓ acquired |
| Logic Pro 11 | ~€230 | ✓ acquired |
| Pixelmator Pro | ~€50 | ✓ acquired |
| **Subtotal software starter** | **~€700** | **✓ done** |

### Already owned or external

- Final Cut Pro 12.2 (on each editor's own Mac)
- X2Pro Audio Convert ✓
- DaVinci Resolve 20 Studio (colorist, Windows)
- Pro Tools (external mixer, France)
- UGreen DXP 8800 Plus NAS + 10GbE infrastructure ✓
- Nomad Windows RTX 5090 × 4 rig ✓
- Tailscale mesh ✓
- R2 Cloudflare off-site backup ✓
- Claude Code subscription ✓
- SpliceKit MCP (open source, patched locally for macOS 26.3) ✓

### Hardware on hold (acquire when available)

| Item | Estimated | When |
|---|---|---|
| Mac Studio M5 Ultra 64-128GB | €5000-8000 | Q3-Q4 2026 when Apple releases |
| iPad Pro M4 13" (HDR preview, optional) | ~€1800 | as budget allows |
| **Subtotal hardware future** | **€5000-10000** | **phased** |

### Recurring costs

| Item | Monthly | Notes |
|---|---|---|
| PostLab | $0 | **Not used** — divided libraries approach instead |
| Tailscale | $0 | existing setup |
| R2 Cloudflare | ~€10-30 | existing setup, tiny |
| Claude Code | existing subscription | already in place |

**Total recurring additional for Goldberg**: negligible.

### Variables to confirm with the colorist

The colorist is expected to provide:
- Their own Windows workstation
- Their own Resolve Studio 20 license (confirmed)
- Their own HDR reference monitor (Flanders, Sony BVM, Eizo CG3146, or Apple Pro Display XDR — confirmed available)

No cost to the production.

### Total post-production software and infrastructure for Goldberg

| Phase | Cost |
|---|---|
| Already spent (Apple ProApps + FCP plugins starter) | **~€700** |
| Already owned and in place (NAS, network, backup, specialist tools, VFX rig) | ~€0 marginal for Goldberg |
| Hardware upgrade path (Mac Studio M5 Ultra when available) | **~€5000-8000** (Q3-Q4 2026) |
| Optional HDR preview (iPad Pro M4) | ~€1800 |
| **Total floor (working setup, ready now)** | **~€700** |
| **Total ceiling (Mac upgrade + HDR preview)** | **~€10500** |

For a 20-month documentary shipping to three delivery standards including Netflix HDR, this is lean. The saving comes from four decisions:
1. No PostLab subscription (divided libraries approach)
2. Reusing existing hardware (NAS, VFX rig, Tailscale, R2) already paid for
3. Delaying the Mac Studio M5 Ultra purchase until the machine actually exists
4. The colorist and the mixer bring their own infrastructure

---

## Trade-offs and known risks

Every architecture makes choices. Here are the trade-offs this one makes, so that the team and the production know what is being accepted and what is being deferred.

### 1. FCP single-vendor lock-in

Final Cut Pro runs only on macOS. If Apple changes the Pro Apps strategy, raises prices dramatically, or introduces incompatible changes in a future version, the whole edit team's tooling is affected. Mitigation: the project keeps ACES working space throughout, keeps all media on the NAS in open formats (BRAW natives, ProRes proxies), and exports to FCPXML 1.10 regularly as an interchange format. If needed, the whole edit could migrate to Resolve or Premiere within a few weeks (painful, but possible).

### 2. SpliceKit is open-source, maintained by one developer

SpliceKit is a hack at Apple's private APIs. It works today because the community patches it (including the macOS 26.3 PR #51 that this team contributed to upstream). It will break with future FCP or macOS updates. Mitigation: the team avoids SpliceKit for mission-critical automation. It is used for accelerating workflows, not for irreversible operations. If SpliceKit breaks, the fallback is the fcpxml-mcp-server path (slower but stable).

### 3. No PostLab means higher discipline cost

The divided libraries approach relies on team discipline, not tooling. If an editor accidentally edits another editor's library, or if the `LOCKS.md` coordination is ignored, conflicts occur. Mitigation: strict naming conventions, a simple Python CLI for locks, and a weekly integration meeting where the lead editor merges contributions into the master library.

### 4. Local Nomad rig is a single point of failure

The Nomad Windows rig with RTX 5090 × 4 is where all heavy AI and Blender work happens. If it fails mid-project, the VFX pipeline is blocked. Mitigation: daily rsync of all Nomad work to the NAS, secondary VFX capability on a less powerful machine (Mac Studio with CPU-only Blender fallback), and a known path to rent cloud GPU (RunPod, Paperspace) if the local rig fails.

### 5. HDR delivery depends on the colorist's equipment

The Netflix HDR master is validated on the colorist's reference monitor. If that monitor fails, or if the colorist becomes unavailable, the HDR delivery is blocked. Mitigation: identify a backup colorist early in the project. Option to rent time in a post house for a single day of HDR mastering if the project becomes urgent.

### 6. AI generation remains experimental for HDR

As of April 2026, the only native HDR AI video model in production is Luma Ray3 (cloud, paid). Open source models (Flux.2, Wan 2.2, LTX-2) output SDR by default. The [Lightricks HDR latent alignment paper](https://arxiv.org/abs/2604.11788) points toward open source HDR generation within months, but it is not yet production-ready. Mitigation: the team uses tone mapping from SDR generation for most AI footage, and reserves Luma Ray3 (cloud, per-generation cost) for hero shots that require native HDR dynamic range.

---

## Related resources in this ecosystem

- [deep-research / the-timeline-becomes-readable](https://github.com/12georgiadis/deep-research/tree/main/10-the-timeline-becomes-readable) — the philosophical essay on the SpliceKit paradigm shift
- [deep-research / the-2026-editing-map](https://github.com/12georgiadis/deep-research/tree/main/11-the-2026-editing-map) — the full NLE benchmark for 2026
- [fcp-workflow](https://github.com/12georgiadis/fcp-workflow) — the reference FCP stack this blueprint is based on
- [film-indexer](https://github.com/12georgiadis/film-indexer) — the multi-LLM editorial council for rush indexing
- [comfyui-cinema-pipeline](https://github.com/12georgiadis/comfyui-cinema-pipeline) — the ComfyUI production pipeline used here
- [SpliceKit (elliotttate fork)](https://github.com/elliotttate/SpliceKit) — the dylib this whole pipeline depends on
- [SpliceKit PR #51](https://github.com/elliotttate/SpliceKit/pull/51) — the macOS 26.3 compatibility fix contributed from this project

---

## Credits and acknowledgments

This architecture was developed in April 2026 by Ismaël Joffroy Chandoutis (director) in conversation with Claude (Anthropic), across a long session that synthesized roughly ten years of practice into a single actionable blueprint for *The Goldberg Variations*. It draws on the work of many people: the SpliceKit community around Elliott Tate, the CommandPost ecosystem around Chris Hocking at LateNite Films, the Color Trix team for Color Finale 2 and BRAW Toolbox, and the ASWF community for OpenTimelineIO and the OCIO config that makes ACES cross-tool work possible at all.

Published under CC BY-SA 4.0 so that other independent documentary teams can adapt it for their own projects.

Questions, corrections, and war stories welcome in issues.

---

*Ismaël Joffroy Chandoutis — Paris, April 2026*
