# Hybrid Film Post-Production Pipeline: AI-Orchestrated 5-Person Team with Final Cut Pro, ComfyUI, and DaVinci Resolve

5-person team, 18-24 months, archival material + AI-generated visuals, 3-platform delivery (festival DCP, streaming, Netflix HDR). FCP 12.2 as hub NLE, ComfyUI on GPU rig for visual generation, Resolve for final grade and IMF delivery, Claude Code as orchestration layer.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HYBRID FILM POST-PRODUCTION                       │
│                    ~20-month pipeline, 3-platform delivery           │
└─────────────────────────────────────────────────────────────────────┘

[ PRODUCTION CAPTURE ]
   Blackmagic Pocket/Cinema 6K → BRAW 12:1 / iPhone ProRes
             │ MHL + xxHash ingest + R2 Cloudflare backup
             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  NAS (8-bay RAID 6, dual 10GbE, NVMe cache)                         │
│  ├── /01_rushes_natives/    → camera originals                      │
│  ├── /02_rushes_proxies/    → ProRes 422 LT 1080p (auto-generated)  │
│  ├── /03_fcp_libraries/     → 5 divided libraries                   │
│  ├── /04_assets/            → LUTs, Motion templates, fonts         │
│  ├── /05_vfx_outputs/       → ComfyUI / Blender renders             │
│  ├── /06_audio/             → music, sound design, Logic sessions   │
│  ├── /07_exports/           → masters + deliverables                │
│  ├── /no-ai-zone/           → protected footage, pipeline excluded  │
│  └── /snapshots/            → 00_MASTER dated snapshots (30d min)   │
└─────────────────────────────────────────────────────────────────────┘
             │                       │                      │
             ▼                       ▼                      ▼
[ EDITING TEAM: 5 MACS ]    [ VFX TEAM: GPU RIG ]    [ COLORIST ]
   FCP 12.2 + SpliceKit MCP    Windows RTX rig           Resolve Studio 20
   + Color Finale 2 Pro        ComfyUI + Flux.2           HDR reference monitor
   + BRAW Toolbox              Blender 4.4                ACES 1.3 ACEScct
   + Motion + Logic Pro        Pallaidium                 CDL export → FCP
   + Compressor + Pixelmator
   Claude Code orchestration   → ProRes 4444 to NAS
   NVMe shuttle (local canon)
             │
             ▼
[ SOUND ]                    [ AI PIPELINE (overnight) ]
   FCP → Logic Pro              film-indexer council (5 voices)
   FCP → X2Pro → Pro Tools      Gemini Flash + Claude Opus
   printmaster return           Pinecone + SQLite
                                → FCPXML injection via SpliceKit
             ▼
[ DELIVERY ]
   ├── DCP festival (P3-D65 SDR, Compressor 4.8)
   ├── Streaming H.264/H.265 (Rec.709, Compressor)
   └── Netflix IMF HDR (HDR10 / Dolby Vision, Resolve Studio)
```

---

## Part I: Philosophy

### The No-AI Zone

Before any AI tool processes your footage, **watch 30 hours of raw rushes alone, without AI assistance**. Designate a folder on your NAS as the No-AI Zone. The pipeline never reads this folder.

A vision model generates labels that reflect its statistical understanding of the world. It names what it recognizes. The shot you actually need -- the one that doesn't look like anything it has seen -- has no tag. If you let the AI catalog your material before you know it yourself, you will spend 20 months searching through the AI's understanding of your film rather than your own.

**Protocol:**
1. Intake new footage to NAS. Pipeline watcher triggers on arrival.
2. Before running the pipeline, move 10-20% of the most significant clips to `NAS:/no-ai-zone/` (manual selection by filmmaker).
3. These clips are excluded from Phase 0-8 processing by path filter.
4. Log them manually in `reports/no-ai-zone-log.md`.
5. Enter them into the pipeline later if you choose -- but only after your own reading is complete.

### The Scratch Library

Create `99_scratch.fcplibrary` that **never merges into `00_MASTER`**. Not versioned, not indexed, not backed up with master rigor. This is the space for liquid writing, failed experiments, half-formed ideas.

Without a scratch space, editors either stop experimenting (losing creative benefit) or contaminate the master with unfinished ideas (losing organizational benefit). The separation must be architectural, not disciplinary.

**Rules:**
- No clip only assembled in `99_scratch` should be referenced in `00_MASTER`.
- The scratch library can be wiped. Accept this.

### Discipline Over 20 Months

The team at month 18 is not the team at month 2. By month 18: one editor may have left and been replaced, discipline around commit protocols will have eroded, the merge assistant will have developed shortcuts that bypass written protocols, the AI pipeline will have encountered at least one major software version that broke a critical component.

**Structural response:**
- Write discipline protocols now as if explaining them to someone who was not in the room when you made the original decisions.
- Schedule protocol review sessions at months 4, 8, 12, 16.
- Editors sign a one-page acknowledgment when joining. Treat protocol reviews as production events.

---

## Part II: Technical Architecture

### Software Stack

#### Primary Creative Chain

| Tool | Version | Role | Cost (indicative) |
|------|---------|------|-------------------|
| Final Cut Pro | 12.2 | Hub NLE | €299 one-time |
| SpliceKit MCP | current | FCP programmatic layer (direct in-process, 200+ tools) | free / open source |
| Motion | 5.8 | Titles, VFX templates, FCP-native | €50 one-time |
| Logic Pro | 11 | Preliminary audio, FCP-native round-trip | €230 one-time |
| Color Finale 2 Pro | current | Primary grade in FCP, ASC CDL export to colorist | ~$150 one-time |
| BRAW Toolbox | current | Native BRAW import in FCP, no transcode step | ~$80 one-time |
| X2Pro Audio Convert | current | AAF export to Pro Tools | ~$200 one-time |
| Compressor | 4.8 | DCP, H.265, deliverable masters | €50 one-time |
| Pixelmator Pro | current | Static graphics, replaces Photoshop | €50 one-time |

#### FCP Ecosystem Plugins

| Plugin | Publisher | Role | Cost |
|--------|-----------|------|------|
| Gyroflow Toolbox | fcp.cafe | Gyroscopic stabilization (reads embedded gyro) | free |
| Marker Toolbox | LateNite Films | Review markers from Vimeo / Frame.io | bundle |
| Fast Collections | LateNite Films | Smart Collections batch generation | bundle |
| CommandPost | community | Hardware panels, batch tagging, Lua automation | free |
| Neat Video | ABSoft | Denoise (FCP plugin, for high-ISO footage) | ~$100 |

#### Grade and Final Mix Chain

| Tool | Version | Role | Location |
|------|---------|------|----------|
| DaVinci Resolve Studio | 20 | Final grade + Netflix IMF wrap | Colorist workstation (Windows) |
| Pro Tools | current | Final mix | External mixer |

#### AI Visual Chain

| Tool | Role |
|------|------|
| ComfyUI (Flux.2, Wan 2.2, LTX-2) | AI-generated visuals, reconstruction sequences, experimental VFX |
| Blender 4.4 + Pallaidium | 3D, AI generation inside VSE |
| Unity 6 LTS | Fallback if Blender is unavailable or insufficient for a specific 3D task |

**Not used**: Adobe Firefly (cloud-dependent subscription), After Effects (Motion + Resolve covers the use cases), CapCut (no professional delivery path), LTX Studio (cloud-dependent), Godot / Unreal (no real-time interactive elements in this pipeline), Resolve as primary NLE (see trade-offs).

#### Claude Code + MCP Layer

| Component | Role |
|-----------|------|
| Claude Code CLI | Agentic orchestration across all tools |
| SpliceKit MCP | Direct in-process FCP control (200+ tools via injected dylib) |
| comfyui-mcp | ComfyUI workflow orchestration from Claude |
| fcp-xml MCP | FCPXML-level batch operations (complement to SpliceKit) |
| Pinecone MCP | Semantic search across rush archive |

#### Trade-offs

**FCP vs. Resolve as primary NLE**: Resolve 20 has a stronger built-in collaboration system. FCP with divided libraries is workable for 2-3 editors; at 5 editors it requires the LOCKS.md protocol and a merge assistant role. Resolve was eliminated as primary NLE because: Apple Silicon performance gap is significant for proxy-heavy workflows; SpliceKit MCP has no Resolve equivalent; FCP-native round-trips with Motion, Logic, and Compressor reduce friction. Resolve remains essential for final grade and Netflix delivery.

**SpliceKit MCP risk**: depends on a patched dylib against private Apple APIs. See Part V for fallback strategy. Do not use SpliceKit for irreversible operations on `00_MASTER`.

**PostLab vs. divided libraries**: PostLab solves the collaboration problem correctly and cleanly. Divided libraries solve it at a lower recurring cost but with higher discipline overhead. At 5 editors over 20 months, divided libraries only hold if a dedicated merge assistant role is funded.

### Hardware Infrastructure

#### Shared Storage (NAS)

| Component | Specification | Notes |
|-----------|--------------|-------|
| Primary NAS | 8-bay RAID 6 (UGreen DXP 8800 Plus or equivalent) | RAID 6 mandatory at 16TB drives: RAID 5 URE rebuild risk is statistically unacceptable |
| NAS drives | 8x 16TB (96 TB usable) | Sized for 10+ TB raw footage + proxies + project files |
| NAS cache | 2x NVMe M.2 (internal cache slots) | Hot files at SSD speed, cold archives on HDD |
| Network | 10GbE switch, Cat 6a or SFP+ for long runs | Required for 5 simultaneous FCP instances on NAS proxies |

**Critical rule: no simultaneous library access.** Five FCP instances breathing simultaneously on the same library database over NAS produces beachballs and background render conflicts. Each editor maintains a local canonical copy of their assigned library on their NVMe shuttle. The NAS copy is the sync destination, not the live working location. Sync happens end of day, not mid-session.

#### Editing Workstations

| Role | Machine | RAM | Notes |
|------|---------|-----|-------|
| Lead editor | Mac Mini M4 (or Mac Studio M4/M5 Ultra) | 24-64GB | Native edit when RAM allows; proxies otherwise |
| Lead editor mobile | MacBook Air M3 | 16GB | Proxies only, travel and review |
| Editors 2-5 | Own Mac (M-series) | 16-32GB | ProRes proxies from NAS |

All 5 editors work on **ProRes 422 LT 1080p proxies** generated automatically at ingest. Not ProRes Proxy (too compressed for fine cut), not 422 HQ (overkill). LT at 1080p is the right balance: trivial to decode on Apple Silicon, instant scrub, quality sufficient for all creative decisions. Native originals are reserved for final export passes.

**NVMe shuttle drives**: one 4TB Thunderbolt 4 drive per editor. Local canonical library during session. End of day: sync to NAS.

#### Off-site Backup

- **Cloudflare R2** (or equivalent S3-compatible): versioned backup of FCPXML, database, and critical masters
- **Tailscale mesh**: secure access to all machines including remote GPU rig, without VPN configuration overhead

#### GPU Compute (Remote Rig)

A remote GPU workstation (RTX 5090 or equivalent, connected via Tailscale) handles:

- BRAW decode at scale (Phase 1)
- Gemini Vision keyframe extraction (Phase 3)
- Audio neural analysis: pyannote, yamnet, openunmix (Phase 4)
- SigLIP visual embeddings (Phase 3)
- ComfyUI AI visual generation

Dispatch via SSH + rsync. `--remote` flag in pipeline commands routes heavy computation to the GPU workstation.

### Team Structure

| Role | Library | Tools | Notes |
|------|---------|-------|-------|
| Lead editor | `00_MASTER.fcplibrary` | FCP + Motion + Logic + Color Finale 2 + Claude Code | Creative decisions only. Not the merge integrator (see 6th Role). |
| Editor 2 -- rushes / ingest | `01_rushes.fcplibrary` | FCP + BRAW Toolbox + CommandPost | Handles native ingest, proxy generation oversight |
| Editor 3 -- archives | `02_archives.fcplibrary` | FCP + custom normalization scripts | Archival material only |
| Editor 4 -- interviews | `03_interviews.fcplibrary` | FCP + native FCP 12 transcription | Interview footage only |
| Editor 5 -- reconstructions / VFX | `04_reconstructions.fcplibrary` | FCP + BRAW Toolbox + Color Finale 2 | Reconstructed sequences, AI-generated inserts |
| **Merge assistant (6th role)** | `00_MASTER.fcplibrary` write access | FCP + fcpxml-mcp-server | Daily integration, snapshots, conflict resolution. Full-time role. |
| VFX team (1-3 people) | Not in FCP tree | ComfyUI + Blender | ProRes 4444 output to NAS `/05_vfx_outputs/` |
| Colorist | External | DaVinci Resolve Studio 20 | FCPXML round-trip, ACES grade, CDL export |
| Sound mixer | External | Pro Tools | AAF via X2Pro, printmaster return |

---

## Part III: Workflow Protocols

### 1. Capture and Ingest

1. Camera cards arrive from shoot.
2. DIT copies to two drives with `ascmhl create -h xxh64` for xxhash integrity verification.
3. Primary copy pushed to NAS `/01_rushes_natives/`.
4. Editor 2 runs FCP import with proxy media enabled (ProRes 422 LT 1080p). Proxies auto-populate `/02_rushes_proxies/`.
5. Nightly cron rsyncs natives + MHL to Cloudflare R2 for off-site backup.

**Proxy workflow with LUT**: generate proxies in **Log color space** (BMD Film Gen 5, Apple Log, or original camera log). Apply a correction LUT via an FCP adjustment layer above the proxy footage. This way: swap the LUT without re-exporting proxies; at conform, FCP relinks BRAW originals by filename and the log proxies become irrelevant.

**Recommended processing order in Phase 1**: RAW decode → stabilization (Gyroflow or Mercalli v6) → denoise (Neat Video) → ProRes 422 LT export.

### 2. Editing (Divided Libraries)

Each editor works strictly within their assigned library. Scope must be **genuinely watertight**: interview footage does not appear in the archives library; archive clips used in reconstruction are imported from the shared read-only assets folder (`/04_assets/`), never duplicated.

**Before first import: the Rolemap**

Audio role contamination (interview audio mislabeled as ambient, VO mixed with dialogue) is one of the most common and difficult-to-repair failure modes in FCPXML-based collaboration. It propagates through imports and is invisible until the sound mix.

Write and distribute the rolemap before any import:

```
Dialogue    → sync interview sound, direct address
Ambient     → room tone, environmental sound, crowd
Music       → score, source music, needle drops
VO          → filmmaker narration, third-party narration
FX          → effects, designed sound
```

All editors sign off on this document. The merge assistant checks role assignments on every FCPXML import. Mislabeled roles are corrected before merge, not after.

**LOCKS.md protocol**: plain JSON file on NAS root. Editors declare working hours and lock clips before each session:

```json
{
  "2026-04-15T09:00:00": {
    "clip_id": "A022-C003",
    "locked_by": "Editor 2",
    "reason": "Archive grading in progress",
    "expected_release": "2026-04-16T10:00:00"
  }
}
```

Locks are released by the merge assistant after successful integration. When two editors have conflicting versions of the same clip, the merge assistant has final authority on which version enters `00_MASTER`.

### 3. The 6th Role

**This role is not optional on a 5-editor project lasting 20 months.**

The lead editor cannot function simultaneously as merge integrator and creative editor. By month 6, integration overhead crowds out editorial clarity. By month 12, the project accumulates merge debt.

**Daily responsibilities:**
- Pull each editor's library updates from NAS, verify no partial writes
- Run pre-merge diff: compare incoming FCPXML against `00_MASTER` snapshot before any import
- Flag conflicts for resolution (authority over which version wins)
- Import approved FCPXML into `00_MASTER`
- Create dated snapshot of `00_MASTER` after each integration session

**Weekly responsibilities:**
- Test fallback tools (see Part V)
- Review LOCKS.md for stale locks
- Check rolemap compliance on recent imports

**Budget implication**: this is a salaried position for the duration of post-production. If budget does not allow it, reduce from 5 editors to 3 with genuinely watertight scope -- do not assign the role informally to the director.

### 4. Snapshots

Before every FCPXML integration into `00_MASTER`, the merge assistant creates a dated snapshot:

```
NAS:/[project]/snapshots/
  2026-04-15T18:30-MASTER.fcpbundle.tar.gz
  2026-04-14T19:15-MASTER.fcpbundle.tar.gz
```

Retention: 30 days minimum on NAS. Older snapshots archived to cold storage (R2), not deleted.

**Why this matters**: FCPXML imports have a known failure mode -- they can break compound clip references and role assignments in ways not immediately visible. Without snapshots, a corrupt import discovered two weeks later has no clean recovery path.

### 5. VFX and AI Generation

VFX team works on the GPU rig, disconnected from the FCP project tree. Their deliverables flow one direction only: NAS → FCP.

Typical cycle:
1. Lead editor identifies a shot needing VFX or AI generation.
2. Claude Code sends a brief to the VFX team via Discord or custom bridge.
3. VFX team runs ComfyUI / Blender workflow on GPU rig.
4. Output renders as ProRes 4444 or EXR to `/05_vfx_outputs/<shot_id>/`.
5. Claude Code detects new files and imports via SpliceKit `import_fcpxml()` or direct media add.
6. Editor places the new clip in the timeline.

Lock ComfyUI custom node versions for each completed VFX sequence. Document which node versions were used. Export finished AI sequences as ProRes masters immediately after approval -- these are the deliverables; the ComfyUI workflow is the process.

### 6. Color Grade (FCP → Resolve → FCP)

1. Export partial FCPXML 1.10 containing only sequences ready for grade.
2. Consolidate associated media. Generate ASC MHL for the package.
3. Deliver to colorist via NAS, cloud drop, or shuttle.
4. Colorist imports into Resolve Studio, conforms, grades in ACES 1.3 ACEScct.
5. Colorist renders graded clips as ProRes 4444 (same source filename + `_GRADED` suffix).
6. Graded files returned to NAS.
7. Lead editor swaps originals for graded clips via SpliceKit batch operations.

**On iteration**: if the cut changes after grade, re-export updated partial FCPXML. Colorist uses Resolve's ColorTrace to reapply existing grades to the new cut. Only new or trimmed clips need fresh grade attention.

### 7. Sound (FCP → Logic Pro → Pro Tools → FCP)

**Preliminary (in-house)**:
1. Build preliminary sound in FCP with audio roles assigned (Dialogue.Sync, Dialogue.Narration, Music.Score, Effects.Ambience, Effects.Hard).
2. For music and complex sound design: `File → Send Audio to Logic Pro`.
3. Bounce stems back to FCP via AIFF/WAV.

**Final mix**:
1. At picture lock, export AAF via X2Pro Audio Convert. Audio roles become Pro Tools tracks automatically.
2. External mixer receives AAF + reference QuickTime ProRes 422 HQ with burn-in.
3. Mixer delivers stems (dialogue, music, effects) + printmaster.
4. Import printmaster into FCP on a new audio track. Mute preliminary audio.

### 8. ACES Working Space

The entire pipeline works in **ACES 1.3 ACEScct** from ingest to delivery:

- Editing team sees SDR ODT (Rec.709) on standard monitors during edit.
- Colorist grades in ACES on HDR reference monitor (Flanders Scientific, Sony BVM, or equivalent).
- Single ACES master trims to each delivery target via different ODTs: P3-D65 SDR for DCP, Rec.709 for streaming, Rec.2020 PQ for Netflix HDR.
- Future-proof: any new delivery format uses the same ACES master.

ACES config works cross-tool: FCP (ACES Wide Gamut HDR), Blender (OCIO ACES config), ComfyUI (appropriate HDR nodes), Resolve (ACEScct timeline).

### 9. Delivery

| Target | Format | Tool | Notes |
|--------|--------|------|-------|
| Festival DCP | JPEG2000 in MXF, SMPTE DCP | Compressor 4.8 | P3-D65 SDR, 5.1 or stereo from printmaster |
| Streaming | H.264 or H.265 MP4 | Compressor 4.8 | Rec.709 SDR, stereo downmix |
| Netflix IMF | IMF 2067-21, HDR10 or Dolby Vision | DaVinci Resolve Studio | One-way operation at end of chain; FCP does not export IMF natively |

**Netflix delivery specs change every 6 months.** Download the Netflix Partner Help Center spec and lock the contractually required version in your delivery contract.

---

## Part IV: AI Logging Pipeline (9 Phases)

Runs overnight. Replaces approximately 9 human roles in the dailies-to-editorial process. Total API cost for a 200-day shoot at ~100 clips/day: approximately $305.

```
[Camera card / NAS intake]
        ↓
[Phase 0]  Inventory + xxhash64 verification
        ↓
[Phase 1]  Ingest + proxy generation (H.264 low / ProRes 422 LT)
        ↓
[Phase 2]  Local transcription (MacWhisper + Parakeet v3, 0 API cost)
        ↓
[Phase 3]  Image analysis (Gemini Flash vision, ~$0.27/day)
        ↓
[Phase 4]  Audio analysis (pyannote + LUFS + yamnet, 0 API cost)
        ↓
[Phase 5]  Narrative detection (Claude Opus 1M context, ~$0.50/day)
        ↓
[Phase 6]  Council review: 5-voice LLM editorial scoring (~$0.50/day)
        ↓
[Phase 7]  FCP injection (SpliceKit MCP + enriched FCPXML v1.11)
        ↓
[Phase 8]  Semantic query interface (Pinecone + SQLite)
```

**Central database**: SQLite at `index/[project].sqlite`.
**Semantic index**: Pinecone (1024 dims, `multilingual-e5-large`), namespaced by day.
**FCPXML output**: enriched v1.11 at `fcpxml/YYYY-MM-DD.fcpxml`.

### Phase 0 -- Inventory and Hash

Scans source volumes, builds JSON inventory with xxhash64 fingerprint, size, codec, framerate, timecode, EXIF creation date. All clips not previously seen are marked `status=new`. **API cost**: zero.

### Phase 1 -- Ingest and Proxy Generation

Two derivatives per clip: H.264 540p for AI vision analysis, ProRes 422 LT 1080p for FCP. RAW decoded via BRAW Toolbox / DaVinci Resolve headless. Processing order: decode → stabilization (Gyroflow) → denoise (Neat Video) → ProRes export. **API cost**: zero.

### Phase 2 -- Local Transcription

MacWhisper (Parakeet v3 engine). Word-level timestamps. Auto-classifies dialogue vs. silence vs. ambient. Speaker diarization against reference voice prints if available. **Rule: local transcription only, never OpenAI Whisper API.** Parakeet v3 runs at ~20x real-time on Apple Silicon. **API cost**: zero.

### Phase 3 -- Image Analysis

N keyframes per clip extracted (1 frame/2s up to 30s, then 1/5s, max 60 frames/clip). Each batch sent to Gemini Flash with structured JSON prompt. Returns: subject, shot size, camera movement, lighting, exposure flag, color temperature estimate, LUT guess, noise level, focus quality, continuity markers, `narrative_hypothesis`, `grade_challenges`, `director_second_look`. Visual embeddings (SigLIP, local on GPU) pushed to Pinecone for similarity search. **API cost**: ~$0.27/day, ~$55/200 days.

**Human checkpoint**: review image analysis report on first day of each new shooting location. Calibrate the prompt if the model systematically miscategorizes your footage type.

### Phase 4 -- Audio Analysis

Per clip: LUFS integrated + momentary (flag < -30 LUFS or > -14 LUFS), pyannote VAD, openunmix music separation, speaker diarization, yamnet background noise classification (wind, traffic, crowd, AC hum), room tone candidates flagged (silence > 2 seconds). Runs in parallel with Phase 3. **API cost**: zero.

### Phase 5 -- Narrative Detection

Claude Opus (1M context window) ingests combined output of Phases 2-4 for an entire shooting day in one call. Returns: scene assignment per clip, narrative beat, possible edit connections, coverage gaps, continuity contradictions. Output: JSON scene map + markdown report at `reports/day-YYYY-MM-DD-narrative.md`. **Read this report before opening FCP. It is a map, not an instruction.** **API cost**: ~$0.50/day.

### Phase 6 -- Council Review

`film-indexer` runs each enriched clip through a multi-voice LLM editorial panel. Each voice scores 1-10 with 2-3 sentences of notes:

- **Voice 1 (classical editing)**: narrative economy, rhythm, classical cutting logic
- **Voice 2 (vérité)**: unscripted moments, authenticity, what cannot be repeated
- **Voice 3 (visual rigor)**: image quality, compositional discipline
- **Voice 4 (investigative)**: factual value, evidentiary weight
- **Voice 5 (poetic)**: texture, off-screen space, duration, silence

Aggregated `council_score` (0-10): >= 7.5 → `select_A`, 5-7.5 → `select_B`, < 5 → `NG`. Voice weights configurable per material type. **Disagreeing with the council is encouraged. Adjust voice weights based on what it consistently misses.** **API cost**: ~$0.50/day.

### Phase 7 -- FCP Injection

Compiles Phases 0-6 into enriched FCPXML v1.11: `<keyword>` per clip, `<marker>` for takes/narrative beats/room tone candidates, `<rating>` (favorite/rejected), `<note>` (council synthesis + `director_second_look`), pre-assigned audio roles from Phase 4, keyword collections by day/scene/score tier.

**Critical rule**: Claude Code reads and proposes. A human executes write operations into `00_MASTER`. Batch clip swaps happen in a staging library, never directly in `00_MASTER`. The AI proposes the FCPXML; a human imports it. **API cost**: zero.

### Phase 8 -- Semantic Query Interface

Natural language queries across the entire rush archive from Claude Code:

```bash
python pipeline/08_query.py "all shots where subject speaks about childhood with 3+ seconds of silence before responding"
python pipeline/08_query.py "exterior day, backlit, person on phone"
python pipeline/08_query.py "room tones longer than 5 seconds"
python pipeline/08_query.py "NG by visual rigor voice but select by poetic voice"
```

Query cascade: Pinecone multi-namespace search → SQL filter → Claude Opus reranking. Output: markdown list with timecodes + SpliceKit command to jump to clip in FCP. **Cost per query**: ~$0.005.

### Pipeline Cost Summary

| Phase | Cost/day | Total (200 days) | Time/day |
|-------|----------|------------------|----------|
| 0 Inventory | $0 | $0 | 12 min |
| 1 Ingest | $0 | $0 | 2h30 |
| 2 Transcription | $0 | $0 | 40 min |
| 3 Image (Gemini Flash) | $0.27 | ~$55 | 2h30 |
| 4 Audio | $0 | $0 | 30 min |
| 5 Narrative (Claude Opus) | $0.50 | ~$100 | 3 min |
| 6 Council | $0.50 | ~$100 | 1h |
| 7 FCPXML | $0 | $0 | 5 min |
| 8 Queries | variable | ~$50 | N/A |
| **Total** | **~$1.30** | **~$305** | **overnight** |

### Human Roles Replaced

| Human role | Replaced by | Phase(s) |
|------------|-------------|----------|
| Script supervisor / continuity | Gemini Flash + transcription | 2, 3 |
| Assistant director (daily report) | Claude Opus narrative | 5 |
| Preliminary picture editor | Multi-voice LLM council | 6 |
| Preliminary sound editor | pyannote + yamnet | 4 |
| Dailies colorist | Gemini exposure flags | 3 |
| Assistant editor | FCPXML builder + SpliceKit | 7 |
| Archivist | Pinecone + SQLite | 2, 3, 8 |
| Script reader | Claude Opus scene detection | 5 |
| Director second look | `director_second_look` + poetic voice | 3, 6 |

---

## Part V: Delivery and Legal

### Budget Blind Spots

These are not optional costs. They will be incurred. The only question is whether they are planned or discovered at the worst moment.

| Budget line | Estimated range | Notes |
|-------------|-----------------|-------|
| E&O insurance | 15-35k EUR | Mandatory for US distribution, Netflix, broadcast sale |
| Archive clearances (screenshots, streams, UGC) | Often exceeds software budget | Negotiate early; retroactive clearance is more expensive |
| External fact-check | 5-15k EUR | Netflix requires it; festivals increasingly require it |
| QC delivery (Netflix IMF, 2-3 iterations typical) | 3-10k EUR per iteration | Budget 3 iterations minimum |
| Final mix (cinema) | 15-40k EUR | Depends on studio, format, duration |
| Music (composer, sync licenses, needle drops) | 10-50k EUR | Clear rights before post-production begins |
| Subtitles, SDH, audio description, closed captions | 5-15k EUR | Per language; required for accessibility and most distributors |
| Long-term storage post-delivery (10 years of RAW) | 2-8k EUR | Often ignored until the drive fails |
| Contingency | 15-20% of total | Not optional |

**Total potential gap between typical production budget and actual delivery cost: 50-150k EUR.** Budget this before post-production begins, not after.

### E&O Insurance and Chain of Title

Required by most US distributors and streaming platforms. Required documentation: chain of title for all owned elements, release forms for all on-camera subjects, clearance log for all third-party materials, fact-check documentation. **Consult a media lawyer before post-production begins.** The time to identify chain-of-title gaps is before you spend 20 months editing.

### Clearance Log

Maintain throughout production and post-production:

```
| Asset type | Source | Date acquired | Rights status | Notes |
|-----------|--------|---------------|---------------|-------|
| Forum screenshot | Archive.org | 2024-03-12 | Pending | Need fair use memo |
| Stream recording | [platform] | 2024-05-01 | Licensed | Agreement in /legal/ |
```

### The 3-Month Buffer

Negotiate a 3-month buffer into your production contract before post-production begins. If the shoot extends by even a few weeks, or you need to return for additional photography at month 14, picture lock slides by 3 months. The colorist-mixer-QC-IMF window compresses from 16 weeks to 8. The buffer costs nothing if you don't need it.

---

## Part VI: Risk Management

### SpliceKit Fallback Strategy

SpliceKit depends on a patched dylib against private Apple APIs. A macOS or FCP update breaking it is not abstract risk -- it is a statistical certainty over 20 months.

**Rule**: at any point in production, you must be able to deliver the film without SpliceKit. Test this now, not in fine cut.

**Fallback**: `fcpxml-mcp-server` (operates via FCPXML import/export, not FCP internals, slower but stable against Apple API changes).

**Weekly fallback test** (performed by merge assistant):
1. Open a mirror project maintained specifically for testing.
2. Perform one representative task using only fcpxml-mcp-server.
3. Verify result matches the SpliceKit result.
4. Log in `logs/fallback-tests.md`.

**SpliceKit failure response:**
1. Switch immediately to fcpxml-mcp-server for Phase 7.
2. Phase 8 `seek_to_time` queries become manual via timecode reference.
3. Notify team within 24 hours.
4. Assess if break is temporary (update pending) or permanent (API removed).
5. If permanent: evaluate PostLab.

### Backup and Restore Testing

**A backup that has not been restore-tested is not a backup.**

**Monthly restore test protocol:**
1. Select a `00_MASTER` snapshot from 30 days prior.
2. Restore to a test machine without the current working media.
3. Verify FCP can open the restored library and locate all media.
4. Log in `logs/restore-tests.md`: date, snapshot restored, time to complete, issues encountered.

This catches: symlinks that only exist on the primary machine, media referenced but never backed up, FCP library database corruption that backed up silently, proxy media purged but still referenced.

### Colorist Redundancy

A single colorist is the most expensive single point of failure. If they become unavailable during fine cut, there is no Netflix HDR delivery path.

**Protocol:**
1. Identify a backup colorist early in post-production.
2. Contract them to grade at least one test sequence (3-5 minutes) within the first 6 months.
3. Verify they can reproduce the look from the primary colorist's CDL files.
4. Keep them informed at 6-month intervals.

"Identified" is not the same as "contracted and tested."

### FCP Single-Vendor Risk

FCP runs only on macOS. Mitigation: maintain ACES working space throughout, keep all media in open formats (BRAW natives, ProRes proxies), export to FCPXML 1.10 regularly. Migration to Resolve or Premiere is painful but possible.

---

## Related Resources

- [deep-research / the-timeline-becomes-readable](https://github.com/12georgiadis/deep-research/tree/main/10-the-timeline-becomes-readable) -- essay on the SpliceKit / AI-orchestrated NLE paradigm shift
- [deep-research / the-2026-editing-map](https://github.com/12georgiadis/deep-research/tree/main/11-the-2026-editing-map) -- full NLE benchmark 2026: FCP, Resolve, Premiere, Avid, Blender VSE
- [fcp-workflow](https://github.com/12georgiadis/fcp-workflow) -- reference FCP stack this blueprint extends
- [film-indexer](https://github.com/12georgiadis/film-indexer) -- multi-LLM editorial council tool (Phase 6)
- [comfyui-cinema-pipeline](https://github.com/12georgiadis/comfyui-cinema-pipeline) -- ComfyUI production pipeline documentation
- [SpliceKit (elliotttate)](https://github.com/elliotttate/SpliceKit) -- the dylib this pipeline's programmatic FCP layer depends on

---

[Ismaël Joffroy Chandoutis](https://ismaeljoffroychandoutis.com/) — Paris, 2026
