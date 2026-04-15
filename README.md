# An Independent Documentary Post-Production Blueprint: AI-Orchestrated 5-Person Pipeline with Final Cut Pro, ComfyUI, and DaVinci Resolve

> **License**: [Creative Commons BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)
> **Version**: 2.0
> **Context**: Long-form documentary post-production, 18-24 months, multi-persona subject, archival material, experimental VFX, 3-platform delivery (festival DCP, streaming, Netflix HDR).
> **Audience**: Independent filmmakers navigating complex documentary post-production without a large studio infrastructure.

---

## Overview

This blueprint documents an AI-orchestrated post-production architecture designed for independent documentary filmmakers working with a small team (3-5 editors), significant archival material, experimental AI-generated visuals, and multi-platform delivery requirements.

The central thesis: an independent filmmaker in 2026 can orchestrate a pipeline that replaces approximately 9 traditional human roles (script supervisor, assistant editor, preliminary picture editor, preliminary sound editor, dailies colorist, archivist, narrative reader, continuity supervisor, second-unit researcher) using a combination of local and API-based AI tools -- provided the filmmaker accepts that the most critical decisions remain irreducibly human.

**What this blueprint covers:**

- The philosophical layer (which must exist before the technical layer)
- Full software stack with honest trade-offs
- Infrastructure for 10+ TB of archival and camera footage
- A 9-phase AI logging pipeline running overnight
- Collaboration protocols for 5 editors using divided libraries
- Delivery, legal, and budget blind spots
- Risk management and fallback strategies

**What this blueprint does not cover:**

- Subject-specific ethical frameworks (see Note on Ethics below)
- Budget negotiation with funders
- Contractual specifics with distributors

**Estimated timeline context**: this document was stress-tested for a 20-month post-production period. Shorter productions will find it over-engineered; longer productions will need to extend the discipline protocols in Part IV.

---

## A Note on Ethics

This document is deliberately silent on the ethics of your specific subject. That silence is not an oversight -- it is structural.

The council review process that validated this blueprint (see Appendix) flagged this with force: a technical architecture without a parallel ethical architecture is not a production plan. It is a production hazard.

**Before using any section of this document, you need a separate ETHICS document of equivalent length covering:**

- Ongoing consent protocols with your subject (at what cadence? what right of withdrawal?)
- The status of third parties affected by your material
- Security of sensitive footage against leaks that could be weaponized
- Psychological exposure of the filmmaker after 20 months in difficult material
- A core ethical question about your film: will this document ask the audience to remain in discomfort, or will it relieve them? This choice is not aesthetic. It determines everything.

The architecture in this document makes no film better or worse ethically. It only makes it faster and more organized.

---

## Part I: Philosophy

### The No-AI Zone

The single most important protocol in this blueprint has nothing to do with software.

Before any AI tool processes your footage, **you must watch 30 hours of raw rushes alone, without AI assistance**. Designate a folder on your NAS as the No-AI Zone. The AI pipeline never reads this folder. You watch this material without tags, without scoring, without pre-selection. You take handwritten notes.

The reason is structural, not sentimental. A vision model trained on billions of images will generate labels that reflect its statistical understanding of the world. It will name what it recognizes. The shot you actually need -- the one that doesn't look like anything it has seen -- has no tag. If you let the AI catalog your material before you know it yourself, you will spend 20 months searching through the AI's understanding of your film rather than your own.

The No-AI Zone is a protected creative space. It is not inefficiency. It is the precondition for the AI to be useful rather than directive.

**Protocol:**

1. Intake new footage into the NAS. The pipeline watcher daemon triggers on new arrival.
2. Before running the pipeline, move 10-20% of the most significant clips to `NAS:/footage/no-ai-zone/` (manual selection by filmmaker).
3. These clips are excluded from Phase 0-8 processing by a simple path filter.
4. They are logged manually by the filmmaker in `reports/no-ai-zone-log.md`.
5. They may be entered into the pipeline later if the filmmaker chooses -- but only after the filmmaker's own reading is complete.

### The Scratch Library

Create a dedicated `99_scratch.fcplibrary` that **never merges into `00_MASTER`**. This library is not versioned, not indexed by the AI pipeline, and not backed up with the same rigor as the master.

This is the space for liquid writing. For trying a cut that you suspect won't work but need to try. For assembling a sequence that feels wrong in 10 directions at once. For experiments that would contaminate the clean state of your master timeline.

The absence of a scratch space is one of the most common failure modes in structured collaborative workflows. Without it, editors either stop experimenting (losing the creative benefit of the collaboration) or start contaminating the master with half-formed ideas (losing the organizational benefit of the structure). The scratch library resolves this tension by making the separation architectural rather than disciplinary.

**Rules:**

- No clip that has only been assembled in `99_scratch` should be referenced in `00_MASTER`.
- The scratch library can be wiped and rebuilt. Accept this.
- Individual editors may maintain their own scratch libraries. They are not synchronized.

### The Moral Center

Before the first cut, write one sentence in your own handwriting describing where your camera stands morally in relation to your subject. Pin it above your editing station.

This sentence is not a logline. It is not a pitch. It is an internal compass. When a cut isn't working and you cannot identify why, re-read the sentence before looking at the technical problem.

This practice is discipline, not mysticism. Editorial decisions accumulate over 20 months. The moral center of a film can drift imperceptibly, cut by cut, until it becomes something its maker did not intend. A written sentence, revisited regularly, functions as a correction mechanism.

The sentence will probably need revision at month 8 or month 12. Revise it deliberately, in discussion with your team. Do not let it drift passively.

### Discipline Over 20 Months

The protocols in this blueprint were designed for a 5-editor team working over 20 months. They were explicitly reviewed by simulating the team dynamics of month 18, not month 2.

**The central insight from that review**: the team at month 18 is not the team at month 2. By month 18:

- One editor may have left and been replaced.
- The discipline around commit protocols will have eroded.
- The merge assistant (see Part III, The 6th Role) will have developed informal shortcuts that bypass the written protocols.
- The AI pipeline will have encountered at least one major software version that broke a critical component.

**Structural response:**

- Write the discipline protocols now as if explaining them to someone who was not in the room when you made the original decisions.
- Schedule explicit protocol review sessions at months 4, 8, 12, and 16.
- The protocols should be living documents. Editors sign a one-page acknowledgment when joining the project.
- Treat the protocol review sessions as production events, not overhead.

---

## Part II: Technical Architecture

### Software Stack

The following stack was selected after extensive review of the 2026 NLE landscape and validated against the specific requirements of complex documentary post-production.

#### Primary Creative Chain

| Tool | Version | Role | Reasoning |
|------|---------|------|-----------|
| Final Cut Pro | 12.2 | Hub NLE | Native Apple Silicon, BRAW Toolbox, SpliceKit MCP, fastest proxy workflow on M-series hardware |
| SpliceKit MCP | current | FCP programmatic layer | Direct in-process control of FCP internals via injected dylib -- enables AI pipeline injection |
| Motion | 5.8 | Titles, VFX templates | FCP-native, no round-trip required |
| Logic Pro | 11 | Preliminary audio assembly | FCP-native round-trip, available on same machine as NLE |
| Color Finale 2 Pro | current | Primary grade in FCP, CDL export | Enables colorist round-trip without leaving FCP for grade review |
| BRAW Toolbox | current | Native BRAW import in FCP | Eliminates transcode step for Blackmagic camera footage |
| X2Pro Audio Convert | current | AAF export to Pro Tools | Standard bridge between FCP and professional audio suites |
| Compressor | 4.8 | DCP, H.265, deliverable masters | Festival DCP (P3-D65 SDR), streaming masters |
| Pixelmator Pro | current | Static graphics | Replace Photoshop; no subscription, M-series native |

#### Grade and Final Mix Chain

| Tool | Version | Role | Location |
|------|---------|------|----------|
| DaVinci Resolve Studio | 20 | Final grade + Netflix IMF wrap | Colorist workstation (Windows or Mac) |
| Pro Tools | current | Final mix | Mix studio (France or remote) |

#### AI Visual Chain

| Tool | Version | Role |
|------|---------|------|
| ComfyUI | current | AI-generated visuals, experimental VFX, reconstruction sequences |
| Blender 4.4 + Pallaidium | current | Backup 3D specific cases |
| Unity 6 LTS | current | Backup real-time interactive elements |
| Godot 4.3 | current | Backup lightweight interactive experiences |

**Tools explicitly excluded**: Adobe Firefly (subscription, cloud-dependent), LTX Studio (cloud-dependent), After Effects (unnecessary if Motion + Resolve covers the use cases), CapCut (consumer-grade, no professional delivery path).

#### Trade-offs (honest assessment)

**FCP vs. Resolve as primary NLE**: Resolve 20 has a stronger built-in collaboration system (multi-user project server). FCP with divided libraries is workable for 2-3 editors with strict discipline; at 5 editors it requires additional tooling (LOCKS.md protocol, merge assistant role). Resolve was eliminated for the primary NLE role because: Apple Silicon performance gap is significant for proxy-heavy workflows; SpliceKit MCP has no Resolve equivalent; FCP-native round-trips with Motion, Logic, and Compressor reduce friction. Resolve remains essential for final grade and Netflix delivery.

**SpliceKit MCP risk**: SpliceKit depends on a patched dylib maintained by a single developer against private Apple APIs. See Part V, SpliceKit Fallback Strategy for the mitigation.

**PostLab vs. divided libraries**: PostLab solves the collaboration problem correctly and cleanly. Divided libraries solve it at a lower recurring cost but with higher discipline overhead. At 5 editors over 20 months, PostLab's recurring cost becomes justifiable; divided libraries become justifiable only if a dedicated merge assistant role is funded (see The 6th Role).

### Infrastructure

#### Storage

| Component | Specification | Reasoning |
|-----------|--------------|-----------|
| Primary NAS | 8-bay RAID 6 (UGreen DXP 8800 Plus or equivalent) | RAID 6 mandatory: at 16TB drives, RAID 5 URE rebuild risk is statistically unacceptable |
| NAS drives | 8x 16TB (96 TB usable in RAID 6) | Sized for 10+ TB raw footage + proxies + project files |
| NVMe shuttle drives | 4 TB Thunderbolt 4, one per editor | Local canonical library during session; synced to NAS end of day |
| Off-site cold storage | Cloudflare R2 (or equivalent S3-compatible) | Versioned backup of FCPXML, database, and critical masters |
| NAS cache | 2x NVMe internal (NAS cache slots) | Metadata reads and proxy access |
| Network | 10GbE switch, Cat 6a or SFP+ for long runs | Required: 5 FCP instances accessing NAS proxies simultaneously |

**Critical rule: no simultaneous library access.** Five FCP instances breathing simultaneously on the same library database over NAS produces beachballs and background render conflicts. The protocol is: each editor maintains a local canonical copy of their assigned library on their NVMe shuttle. The NAS copy is the sync destination, not the live working location. Sync happens end of day, not mid-session.

#### Network

- **Tailscale mesh**: enables the AI pipeline to dispatch compute tasks to a remote GPU workstation (RTX-class, 4+ VRAM for ComfyUI) via SSH without VPN configuration overhead.
- **10GbE local switch**: required for NAS throughput at 5 simultaneous editors.

#### GPU Compute

A remote GPU workstation (RTX 5090 or equivalent, connected via Tailscale) handles:

- BRAW decode at scale (Phase 1 pipeline)
- Gemini Vision keyframe extraction (Phase 3)
- Audio neural analysis (pyannote, yamnet, openunmix) (Phase 4)
- SigLIP visual embeddings (Phase 3)
- ComfyUI AI visual generation

Dispatch is handled by the pipeline orchestrator via SSH + rsync. The `--remote` flag in pipeline commands routes heavy computation to the GPU workstation.

### AI Logging Pipeline (9 Phases)

The pipeline replaces approximately 9 human roles in the dailies-to-editorial process. It runs overnight. The filmmaker reads reports in the morning. Total API cost for a 200-day shoot at approximately 100 clips per day: approximately $305.

```
[Camera card / NAS intake]
        |
        v
[Phase 0] Inventory + hash verification
        |
        v
[Phase 1] Ingest + proxy generation (ffmpeg, H.264 low / ProRes Proxy mezzanine)
        |
        v
[Phase 2] Local transcription (MacWhisper + Parakeet v3)
        |
        v
[Phase 3] Image analysis (Gemini Flash vision + scene detection)
        |
        v
[Phase 4] Audio analysis (pyannote + LUFS + spectral classification)
        |
        v
[Phase 5] Narrative detection (Claude Opus 1M context)
        |
        v
[Phase 6] Council review (multi-voice LLM editorial scoring)
        |
        v
[Phase 7] FCP injection (SpliceKit MCP + enriched FCPXML)
        |
        v
[Phase 8] Semantic query interface (Pinecone + SQLite)
```

**Central database**: SQLite at `index/[project].sqlite` with tables for clips, takes, shots, days, audio analysis, image analysis, and council notes.

**Semantic index**: Pinecone index (1024 dims, `multilingual-e5-large` model), namespaced by day, with separate namespaces for transcripts, visual embeddings, and narrative summaries.

**FCPXML output**: enriched v1.11 FCPXML stored at `fcpxml/YYYY-MM-DD.fcpxml`.

#### Phase 0 -- Inventory and Hash

**Human role replaced**: DIT / shooting archivist.

Scans source volumes, builds a JSON inventory with xxhash64 fingerprint, size, codec, framerate, timecode, EXIF creation date, and embedded LUT name for RAW camera files. All clips not previously seen by the pipeline are marked `status=new`.

```bash
python pipeline/00_inventory.py \
  --source /Volumes/[NAS-Footage-Volume] \
  --db index/[project].sqlite \
  --since [YYYY-MM-DD]
```

**Human checkpoint**: review the inventory report. Validate day groupings before proceeding.

**Estimated time**: ~12 minutes per 2 TB over 10GbE. **API cost**: zero.

#### Phase 1 -- Ingest and Proxy Generation

**Human role replaced**: assistant editor for media preparation.

Generates two derivatives per new clip: a low-res H.264 proxy (540p, 2 Mbps) for AI vision analysis, and a ProRes Proxy mezzanine (1080p) for FCP. RAW camera files are decoded via CLI tools (BRAW Toolbox, DaVinci Resolve headless); ProRes sources pass directly through ffmpeg.

**Optional stabilization and denoise** (applied before mezzanine export):

- Gyroflow (open source, reads embedded gyro data from Blackmagic cameras) for handheld stabilization
- Neat Video (FCP plugin) for high-ISO noise reduction

Recommended order for Phase 1: RAW decode -> stabilization -> denoise -> ProRes Proxy export.

```bash
python pipeline/01_ingest.py \
  --db index/[project].sqlite \
  --proxy-root /Volumes/NAS/[project]/proxies \
  --mezzanine-root /Volumes/NAS/[project]/mezzanine \
  --workers 6 \
  --remote [gpu-workstation-hostname]
```

**Estimated time**: ~2h30 per shooting day (80 RAW + 40 iPhone clips, ~600 GB) overnight on GPU workstation. **API cost**: zero.

#### Phase 2 -- Local Transcription

**Human role replaced**: script supervisor for dialogue, VO, take numbers.

Every clip passes through MacWhisper (Parakeet v3 engine). Audio is extracted as mono 16 kHz WAV. The transcription JSON (word-level timestamps) is processed to:

1. Extract take/scene markers via regex
2. Classify segments as dialogue vs. silence vs. ambient by word-per-second ratio
3. Auto-tag recurring speaker identities against a reference voice print (if available)
4. Store transcription in SQLite and embed sentence-level vectors into Pinecone

**Rule**: local transcription only. No OpenAI Whisper API. Parakeet v3 runs at approximately 20x real-time on Apple Silicon M-series hardware.

```bash
python pipeline/02_transcribe.py \
  --db index/[project].sqlite \
  --engine macwhisper-parakeet-v3 \
  --lang auto \
  --out transcripts/
```

**Estimated time**: ~40 minutes for 10 hours of audio on MacBook M3. **API cost**: zero.

#### Phase 3 -- Image Analysis

**Human role replaced**: script supervisor (shot description), preliminary colorist, preliminary picture editor.

For each clip, N keyframes are extracted (1 frame per 2 seconds up to 30 seconds, then 1 per 5 seconds beyond that, max 60 frames per clip). Each batch is sent to Gemini Flash via Google API with a structured JSON prompt.

**The prompt asks the model to return**:

```json
{
  "subject": "primary visible subject",
  "secondary_subjects": [...],
  "composition": "dominant rule (thirds, center, symmetry, depth)",
  "shot_size": "ECU|CU|MCU|MS|MLS|LS|ELS",
  "camera_movement": "static|pan|tilt|handheld|gimbal|dolly|zoom|complex",
  "lighting": "natural_day|natural_night|mixed|tungsten|practical|dark",
  "exposure_flag": "ok|underexposed|overexposed|clipping_highlights|crushed_blacks",
  "color_temperature_estimate_k": 5600,
  "lut_guess": "Rec709|LogC|BMDFilm|Generic",
  "location_type": "interior|exterior|car|studio",
  "noise_level": "clean|moderate|heavy",
  "focus_quality": "sharp|soft|oof",
  "continuity_markers": ["visible objects useful for match cuts"],
  "narrative_hypothesis": "what this shot says in one sentence",
  "grade_challenges": ["items the colorist will need to address"],
  "director_second_look": "what works / what does not work, 2 sentences max"
}
```

The `director_second_look` field is the AI's equivalent of the filmmaker's own end-of-day review. It is not trusted blindly -- it is a prompt for the filmmaker's attention, not a verdict.

Visual embeddings (SigLIP model, local on GPU workstation) are pushed to Pinecone for visual similarity search.

**Estimated time**: ~2-3 hours for 120 clips. **API cost**: ~$0.27/day, ~$55 over a 200-day shoot.

**Human checkpoint**: review the image analysis report on the first day of each new shooting location. Calibrate the prompt if the model is systematically miscategorizing your footage type.

#### Phase 4 -- Audio Analysis

**Human role replaced**: preliminary sound editor.

For each clip's audio track:

1. **Loudness**: LUFS integrated and momentary via ffmpeg ebur128. Flag if < -30 LUFS (too quiet) or > -14 LUFS (hot).
2. **Spectral content**: voice activity detection (pyannote VAD), music separation (openunmix on GPU workstation), usable silence detection (room tone > 2 seconds).
3. **Speaker diarization**: pyannote speaker-diarization-3.1. Match against reference voice prints if available.
4. **Background noise classification**: yamnet (TensorFlow Hub) for wind, traffic, indoor quiet, crowd, AC hum, handling noise.
5. **Room tone candidates**: silence windows > 2 seconds, flagged for the sound editor.

```bash
python pipeline/04_audio_analyze.py \
  --db index/[project].sqlite \
  --remote [gpu-workstation-hostname] \
  --room-tone-min-duration 2.0
```

**Estimated time**: ~30 minutes per shooting day, run in parallel with Phase 3. **API cost**: zero.

#### Phase 5 -- Narrative Detection

**Human role replaced**: script reader, dramaturg.

Claude Opus (1M context window) ingests the combined output of Phases 2, 3, and 4 for an entire shooting day in a single call. With a 1M context window, a full day of footage metadata can be processed in one pass.

**The prompt asks the model to identify**:

1. The narrative scene each clip belongs to (named)
2. The narrative beat (setup, development, turn, release)
3. Possible edit connections with other clips from the same day
4. Coverage gaps (what is missing to cut the scene)
5. Continuity contradictions detected

Output: a JSON scene map and a readable markdown report at `reports/day-YYYY-MM-DD-narrative.md`.

**This report is read by the filmmaker every morning**. It is not an instruction. It is a map. The filmmaker decides what to follow and what to discard.

**Estimated time**: ~3 minutes per day (1 API call). **API cost**: ~$0.50/day (approximately 200k tokens input + 20k output on Opus).

**Human checkpoint**: mandatory. Read this report before opening FCP.

#### Phase 6 -- Council Review

**Human role replaced**: second-look director, first editor, artistic advisor.

The `film-indexer` tool runs each enriched clip through a multi-voice LLM editorial panel. Each voice scores the clip from 1 to 10 and writes 2-3 sentences of editorial notes.

**Example editorial voices and their criteria**:

- **Voice 1 (classical editing)**: narrative economy, rhythm, classical cutting logic
- **Voice 2 (documentary verité)**: unscripted moments, authenticity, what cannot be repeated
- **Voice 3 (visual rigor)**: image quality, compositional discipline, what should be cut without sentiment
- **Voice 4 (investigative)**: factual value, evidentiary weight, what cannot be omitted
- **Voice 5 (poetic)**: texture, off-screen space, duration, what works as silence

Voice weights can be adjusted per material type in the configuration file. For interview footage, investigative and verité voices carry higher weight. For texture and ambient footage, the poetic voice carries higher weight.

An aggregator (Claude Opus) combines scores into a `council_score` (0-10). Clips above 7.5 become `select_A`, between 5 and 7.5 `select_B`, below 5 are marked `NG` (no good). These ratings are written into the SQLite database and translated to FCP-compatible ratings (favorite/rejected) in Phase 7.

**Estimated time**: ~1 hour per day. **API cost**: ~$0.50/day, ~$100 over 200 days.

**Human checkpoint**: read the council report. Disagreeing with the council is encouraged. Adjust voice weights based on what the council consistently misses.

#### Phase 7 -- FCP Injection

**Human role replaced**: complete assistant editor function.

Compiles output from Phases 0-6 into enriched FCPXML v1.11:

- `<keyword>` per clip (image tags + audio tags from Phases 3-4)
- `<marker>` for key moments (takes, narrative beats, room tone candidates)
- `<rating>`: favorite (council score >= 7.5) or rejected (council score < 5)
- `<note>`: council synthesis + `director_second_look` text
- Pre-assigned roles (audio role assignment based on Phase 4 speaker diarization)
- Automatic keyword collections by day, scene, and score tier

Two injection modes:

1. **FCPXML direct**: build a new FCPXML from the database and import into FCP.
2. **SpliceKit MCP**: enrich an open FCP library directly (batch actions on existing clips, role assignment, marker placement).

**Critical rule**: Claude Code reads and proposes. A human executes write operations into `00_MASTER`. Batch clip swaps involving graded material happen in a staging library, never directly in `00_MASTER`. The AI proposes the FCPXML; a human imports it.

**Estimated time**: ~5 minutes per day. **API cost**: zero.

#### Phase 8 -- Semantic Query Interface

**Human role replaced**: the filmmaker's own long-term memory.

With Pinecone populated by Phases 2, 3, 5, and 6, the filmmaker can query the entire rush archive in natural language from Claude Code.

**Example queries**:

```bash
python pipeline/08_query.py "all shots where the subject speaks about their childhood with more than 3 seconds of silence before responding"
python pipeline/08_query.py "exterior day shots, backlit, person on phone"
python pipeline/08_query.py "usable room tones longer than 5 seconds"
python pipeline/08_query.py "takes marked NG by the visual rigor voice but select by the poetic voice"
```

The query engine uses a cascade: Pinecone multi-namespace search -> SQL filter -> Claude Opus reranking. Output is a markdown list with timecodes and a SpliceKit command to jump to the clip in FCP.

**Time per query**: 3-10 seconds. **API cost per query**: approximately $0.005.

#### Pipeline Orchestration

Wrapper script `pipeline/run_day.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
DAY="${1:?YYYY-MM-DD date required}"

python pipeline/00_inventory.py --since "$DAY" --until "$DAY"
python pipeline/01_ingest.py --day "$DAY" --remote [gpu-workstation-hostname]
python pipeline/02_transcribe.py --day "$DAY"
python pipeline/03_image_analyze.py --day "$DAY" --model gemini-flash
python pipeline/04_audio_analyze.py --day "$DAY" --remote [gpu-workstation-hostname]
python pipeline/05_script_detection.py --day "$DAY" --model claude-opus
python pipeline/06_council_review.py --day "$DAY"
python pipeline/07_fcpxml_build.py --day "$DAY"
```

Trigger: a LaunchAgent (macOS) or systemd timer (Linux) watches the intake folder. New material that has been stable for 10 minutes triggers the pipeline. The filmmaker receives a notification (Telegram, system notification, or similar) when Phase 5 and Phase 6 reports are ready.

#### Pipeline API Cost Summary

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

#### Human Roles Replaced by the Pipeline

| Human role | Replaced by | Phase(s) |
|------------|-------------|----------|
| Script supervisor / continuity | Gemini Flash vision + transcription regex | 2, 3 |
| Assistant director (daily report) | Claude Opus narrative reader | 5 |
| Preliminary picture editor | Multi-voice LLM council | 6 |
| Preliminary sound editor | pyannote + librosa + yamnet | 4 |
| Dailies colorist | Gemini exposure flags + probe LUT | 3 |
| Assistant editor | FCPXML builder + SpliceKit injection | 7 |
| Archivist | Pinecone + SQLite | 2, 3, 8 |
| Script reader | Claude Opus scene detection | 5 |
| Director second look | `director_second_look` field + poetic voice | 3, 6 |

One human remains in the loop: the director, to read the Phase 5 and Phase 6 reports each morning, and to open FCP once Phase 7 is complete.

---

## Part III: Collaboration Protocols

### Library Management (5 Editors, Divided Libraries)

The divided library approach assigns each editor a specific thematic or temporal scope. Editors work in their assigned library; the merge assistant integrates changes into `00_MASTER`.

**Example division for a multi-persona documentary** (adapt to your structure):

| Library | Scope | Editor |
|---------|-------|--------|
| `00_MASTER.fcplibrary` | Locked master timeline | Merge assistant only |
| `01_interview.fcplibrary` | Direct interview footage | Editor 1 |
| `02_archives.fcplibrary` | Archival material | Editor 2 |
| `03_reconstruction.fcplibrary` | Reconstruction sequences | Editor 3 |
| `04_vfx.fcplibrary` | ComfyUI-generated visuals | Editor 4 |
| `99_scratch.fcplibrary` | Liquid writing space | All (not merged) |

**The scope must be genuinely watertight.** Interview footage should not appear in the archives library. Archive clips used in a reconstruction sequence should be imported from a shared read-only assets folder, not duplicated. Duplication of clips across libraries is the primary source of media location conflicts and version divergence.

**Shared read-only assets** (`/04_assets/` on NAS): LUT files, Motion templates, graphics, music cue references. This folder is version-controlled separately. Changes are submitted as internal pull requests and applied by the merge assistant. No editor writes directly to `/04_assets/`.

**Before first import: the Rolemap**

Audio role contamination -- where interview audio ends up mislabeled as ambient, or VO is mixed with dialogue -- is one of the most common and difficult-to-repair failure modes in FCPXML-based collaboration. It propagates through imports and is nearly invisible until the sound mix.

**Fix it before it happens.** Write and distribute the audio rolemap to all editors before the first import. Pin it in your communication tool.

Example rolemap (adapt to your project):

```
Dialogue    → sync interview sound, direct address
Ambient     → room tone, environmental sound, crowd
Music       → score, source music, needle drops
VO          → filmmaker narration, third-party narration
FX          → effects, designed sound
```

All editors sign off on this document. The merge assistant checks role assignments on every FCPXML import. Mislabeled roles are corrected before merge, not after.

### The 6th Role

**This role is not optional on a 5-editor project lasting 20 months.**

The lead editor cannot function simultaneously as merge integrator and creative editor. By month 6, the integration overhead will crowd out the editorial clarity that the lead is supposed to provide. By month 12, the project will be accumulating merge debt. By month 18, a fine cut that should have been achievable will be delayed by weeks of technical remediation.

**The Merge Assistant** is a full-time role, not a part-time one:

**Daily responsibilities:**

- Run end-of-day sync: pull each editor's library updates from NAS, verify no partial writes
- Run pre-merge diff: compare incoming FCPXML against `00_MASTER` snapshot before any import
- Flag conflicts (two editors have worked on versions of the same archive clip) for human resolution
- Import approved FCPXML updates into `00_MASTER`
- Create dated snapshot of `00_MASTER` after each integration session

**Weekly responsibilities:**

- Verify fallback tools (see Part V: Fallback Strategy)
- Review LOCKS.md for stale locks
- Protocol review: are editors following the rolemap and scope rules?

**Budget implication**: this is a salaried position for the duration of post-production. If budget does not allow it, the correct response is to reduce from 5 editors to 3 editors with genuinely watertight scope boundaries -- not to assign the role informally to the director.

### Sync and Merge Workflow

#### Daily Protocol

1. Each editor works in their local library (NVMe shuttle). NAS copy is not the live working copy.
2. End of day: editor syncs their local library to NAS (rsync, or dedicated sync tool).
3. Merge assistant pulls from NAS, runs pre-push diff, verifies library integrity.
4. Merge assistant creates a dated `00_MASTER` snapshot before importing.
5. Merge assistant imports FCPXML from each editor's library into `00_MASTER`.
6. Conflicts are flagged in LOCKS.md and resolved before the next working day.

#### LOCKS.md Protocol

`LOCKS.md` is a plain JSON file on the NAS:

```json
{
  "2026-04-15T18:30:00": {
    "clip_id": "A022-C003",
    "locked_by": "Editor 2",
    "reason": "Archive grading in progress",
    "expected_release": "2026-04-16T10:00:00"
  }
}
```

Any editor who needs to work on a clip that appears in another library checks LOCKS.md first. Locks are released by the merge assistant after successful integration. When two editors have worked on conflicting versions of the same clip, the merge assistant has final authority on which version enters `00_MASTER`.

#### Snapshots

The merge assistant creates dated snapshots of `00_MASTER` before every FCPXML integration:

```
NAS:/[project]/snapshots/
  2026-04-15T18:30-MASTER.fcpbundle.tar.gz
  2026-04-14T19:15-MASTER.fcpbundle.tar.gz
  ...
```

Retention: 30 days minimum. The snapshot represents the only reliable rollback mechanism if an FCPXML import corrupts the timeline. Snapshots older than 30 days are archived to cold storage (R2), not deleted.

**FCPXML imports have a known failure mode: they can break compound clip references and role assignments in ways that are not immediately visible.** Without snapshots, a corrupt import discovered two weeks later has no clean recovery path.

---

## Part IV: Delivery and Legal

### Budget Blind Spots

This section addresses the budget items most commonly absent from independent documentary production plans. These are not optional costs. They are costs that will be incurred; the only question is whether they are planned or discovered at the worst possible moment.

| Budget line | Estimated range | Notes |
|-------------|-----------------|-------|
| E&O insurance | 15-35k EUR | Mandatory for US distribution, Netflix, broadcast sale |
| Archive clearances (screenshots, streams, user-generated content) | Often exceeds software budget | Negotiate early; retroactive clearance is more expensive |
| External fact-check | 5-15k EUR | Netflix requires it; documentary festivals increasingly require it |
| QC delivery (Netflix IMF, 2-3 iterations typical) | 3-10k EUR per iteration | Budget for 3 iterations minimum |
| Final mix (cinema) | 15-40k EUR | Depends on studio, mix format, and duration |
| Music (composer, sync licenses, needle drops) | 10-50k EUR | Highly variable; clear rights before post-production begins |
| Subtitles, SDH captions, audio description, closed captions | 5-15k EUR | Per language; required for accessibility compliance and most distributors |
| Long-term storage post-delivery (10 years of RAW) | 2-8k EUR | Often ignored until the drive fails |
| Director time as producer-orchestrator | Not typically budgeted | This is a real cost absorbed by the filmmaker |
| Contingency | 15-20% of total | Not optional |

**Total potential gap between typical documentary budget and actual delivery cost**: 50-150k EUR, depending on scope and distribution ambition. Budget this before post-production begins, not after.

### E&O Insurance and Chain of Title

E&O (Errors and Omissions) insurance is required by most US distributors and streaming platforms. It covers legal claims arising from the content of the film. Without it, you cannot sell the film to these markets.

**Required documentation for E&O application:**

- Chain of title for all owned elements (script, footage, music)
- Release forms for all on-camera subjects and contributors
- Clearance log for all third-party materials (archives, music, trademarks, web screenshots)
- Fact-check documentation (who reviewed factual claims and when)
- Errors and omissions memo from production lawyer

**Do not begin post-production without consulting a media lawyer.** The time to identify chain-of-title gaps is before you spend 20 months editing. Retroactive clearances are expensive. Some cannot be obtained.

### Clearance Log

Maintain a structured clearance log throughout production and post-production:

```
| Asset type | Source | Date acquired | Rights status | Cleared by | Notes |
|-----------|--------|---------------|---------------|-----------|-------|
| Forum screenshot | Archive.org | 2024-03-12 | Pending | [lawyer] | Need fair use memo |
| Stream recording | [platform] | 2024-05-01 | Licensed | [rights holder] | Agreement in /legal/ |
```

This log is a living document, maintained by the producer or an appointed clearance coordinator throughout post-production.

### Fact-Check Protocol

For documentaries making factual claims (dates, quotes, legal records, biographical information):

1. A designated fact-checker reviews all factual claims before picture lock.
2. The fact-check process is documented (what was checked, what sources were used, what was changed).

Netflix and other streamers increasingly require this documentation as a delivery requirement. Building the protocol into your production workflow from the beginning is far less disruptive than retrofitting it at delivery.

### Delivery Specifications

Plan for three delivery formats from the beginning; the post-production pipeline should be built around all three simultaneously, not retrofitted for each in sequence.

| Format | Specification |
|--------|--------------|
| Festival DCP | P3-D65 SDR, via Compressor 4.8 |
| Streaming master | H.264 and H.265, Rec.709, via Compressor |
| Netflix HDR | IMF package (SMPTE ST 2067), HDR10 or Dolby Vision, via DaVinci Resolve Studio |

**The master grade strategy**: establish a single ACES 1.3 ACEScct master in DaVinci Resolve. Trim each delivery output via the appropriate ODT (Output Display Transform). This is the correct approach for multi-platform delivery; it avoids regrading from scratch for each deliverable.

**Netflix delivery specifications change every 6 months.** Download the Netflix Partner Help Center delivery specification and lock the contractually required version in your delivery contract. Do not rely on what you knew from your last project.

### The 3-Month Buffer

**Negotiate a 3-month buffer into your production contract before post-production begins.**

This buffer exists because: if the shoot extends by even a few weeks, or if you need to return for additional photography at month 14, your picture lock slides by 3 months. The colorist-mixer-QC-IMF window, which should be 16 weeks, compresses to 8. At 8 weeks, either the quality suffers or the cost doubles (or both).

The buffer costs nothing if you don't need it. It costs significantly more to negotiate it after the fact.

---

## Part V: Risk Management

### SpliceKit Fallback Strategy

SpliceKit depends on a patched dylib against private Apple APIs. The risk: a macOS update or FCP update could break the dylib at any point. This is not an abstract risk; it is a statistical certainty over a 20-month post-production.

**The mitigation is not to avoid SpliceKit. The mitigation is to never be dependent on it for delivery.**

**Rule**: at any point in production, you must be able to deliver the film without SpliceKit. This scenario must be tested now, not in fine cut.

**Fallback: fcpxml-mcp-server** (a separate MCP server that operates via FCPXML import/export rather than direct FCP internals). It is slower and less capable than SpliceKit but does not depend on Apple private APIs.

**Weekly fallback test protocol** (performed by the merge assistant):

1. Open a mirror project (a copy of the current rough cut, maintained specifically for testing).
2. Attempt to perform one representative task using only fcpxml-mcp-server (no SpliceKit).
3. Verify the result matches the SpliceKit result.
4. Log the test result in `logs/fallback-tests.md`.

If the test fails: diagnose before the following week. Do not accumulate untested fallback debt.

**SpliceKit failure response plan** (written and distributed to team):

1. Switch immediately to fcpxml-mcp-server for all pipeline injection (Phase 7).
2. All SpliceKit-specific queries (Phase 8 `seek_to_time`) become manual via timecode reference.
3. Notify team within 24 hours of detection.
4. Assess whether the break is temporary (update pending) or permanent (API removed).
5. If permanent: evaluate PostLab or alternative collaboration tools.

### Backup and Restore Testing

**A backup that has not been restore-tested is not a backup.**

The 3-2-1 rule (3 copies, 2 different media, 1 off-site) is necessary but not sufficient. What you need is a documented, tested restore procedure.

**The restore test protocol:**

1. Once per month, select one `00_MASTER` snapshot from 30 days prior.
2. Attempt to restore it to a test machine that does not have the current working media.
3. Verify that FCP can open the restored library and locate the media.
4. Log the test result in `logs/restore-tests.md`: date, snapshot restored, time to complete, issues encountered.

**What the restore test catches** (that backup monitoring misses):

- Symlinks that point to paths that only exist on the primary machine
- Media that is referenced in the library but was never backed up
- FCP library database corruption that backed up silently
- Proxy media that was purged but is still referenced

If a restore test fails, you have discovered a backup gap before it became a disaster. Fix it immediately.

### Colorist Redundancy

A single colorist is the most expensive single point of failure in documentary post-production. If the colorist becomes unavailable during fine cut -- illness, conflict with production, career change -- you have no Netflix HDR delivery path.

**Protocol:**

1. Identify a backup colorist early in post-production.
2. Contract the backup colorist to grade at least one test sequence (3-5 minutes) on the project within the first 6 months.
3. Verify they can reproduce the look from the primary colorist's CDL files.
4. Keep the backup colorist informed of the project's progress at 6-month intervals.

"Identified" is not the same as "contracted and tested." The test sequence contract is the minimum acceptable standard.

### ComfyUI Integration Risk

AI-generated visuals (reconstruction sequences, experimental material) are one of the most fragile elements in a documentary post-production. The risk profile:

- The GPU workstation running ComfyUI is a single point of failure for all AI visual generation.
- ComfyUI custom node updates can break workflows without warning.
- AI model providers can change API terms or retire models.

**Mitigations:**

- Lock ComfyUI custom node versions for each completed VFX sequence. Document which node versions were used.
- Export finished AI visual sequences as ProRes masters immediately after approval. These masters are the deliverable; the ComfyUI workflow is the process.
- Test all ComfyUI workflows on the backup GPU workstation (if available) before beginning significant generation.
- Budget for regeneration: assume 20% of approved AI visual sequences will need to be regenerated due to technical failures.

---

## Appendix: Council Review Process (LLM-Based Editorial Review)

The council review process used to develop and validate this blueprint is available as a standalone methodology for documentary development.

### What It Is

A structured multi-voice LLM review session in which a document (script, blueprint, rough cut description) is submitted to several distinct AI editorial personas, each applying a different critical framework. The goal is not consensus but coverage: identifying blind spots that a single reviewer would miss.

### The Council for This Blueprint

This blueprint was reviewed by three editorial personas:

**Persona 1 (Ethical filmmaker / process observer)**: focused on what the document does not say about the human and ethical dimensions of the work. Identified the No-AI Zone, the Scratch Library, the Moral Center, and the need for a separate ethics document.

**Persona 2 (Operational editor / long-term production veteran)**: focused on what will break in production practice. Identified the audio rolemap failure, the 6th Role gap, the library sync protocol problem, the SpliceKit fragility, and the importance of the no-AI write rule.

**Persona 3 (Producer / financier perspective)**: focused on what is missing from the business and legal layer. Identified the budget blind spots, the E&O gap, the colorist single-point-of-failure, the clearance log absence, and the timeline buffer.

### How to Run a Council Review

1. Write the document you want reviewed.
2. For each council persona, write a 200-400 word system prompt defining their critical framework, background, and the specific question they are asked to answer (e.g., "What is the most important thing this document does not address?").
3. Submit the document to each persona independently. Do not let one persona's output influence another.
4. Read all responses before synthesizing.
5. Synthesize convergent findings (items multiple personas flagged independently) as the highest-priority issues.
6. Revise the document. Repeat if the revision is substantial.

### Recommended Persona Frameworks for Documentary Projects

- **Process observer**: a filmmaker known for rigorous ethical documentation practices (think Frederick Wiseman's relationship to consent, or Errol Morris's relationship to reconstruction)
- **Operational editor**: a veteran editor of long-form documentary with experience in collaborative multi-editor workflows
- **Producer**: a producer known for navigating complex rights and delivery requirements
- **Distributor**: a documentary acquisitions executive focused on what makes a film deliverable and insurable
- **Subject advocate**: a critic or ethicist who advocates for the subjects of documentary filmmaking

The specific personas you choose should be calibrated to the blind spots most likely in your particular project. A technically rigorous filmmaker may need a stronger producer voice. A producer-filmmaker may need a stronger process voice.

### Limitations

The council review process has known failure modes:

- LLM personas tend toward comprehensiveness. They will flag things that are actually covered elsewhere. Read critically.
- A persona's critique reflects the training data behind it. It may not know about recent changes in delivery specifications, insurance requirements, or software capabilities. Verify independently.
- Consensus across personas does not guarantee correctness. It increases confidence, not certainty.

The council review is a structured method for identifying blind spots. It does not replace legal counsel, production experience, or your own judgment.

---

## Related Resources

- [deep-research / the-timeline-becomes-readable](https://github.com/12georgiadis/deep-research/tree/main/10-the-timeline-becomes-readable) — philosophical essay on the SpliceKit / AI-orchestrated NLE paradigm shift
- [deep-research / the-2026-editing-map](https://github.com/12georgiadis/deep-research/tree/main/11-the-2026-editing-map) — full NLE benchmark for 2026 (FCP, Resolve, Premiere, Avid, Blender VSE)
- [fcp-workflow](https://github.com/12georgiadis/fcp-workflow) — reference FCP stack this blueprint extends
- [film-indexer](https://github.com/12georgiadis/film-indexer) — multi-LLM editorial council tool for rush indexing (Phase 6)
- [comfyui-cinema-pipeline](https://github.com/12georgiadis/comfyui-cinema-pipeline) — ComfyUI production pipeline documentation
- [SpliceKit (elliotttate)](https://github.com/elliotttate/SpliceKit) — the dylib this pipeline's programmatic FCP layer depends on

---

## Credits

Developed in April 2026 by [Ismaël Joffroy Chandoutis](https://ismaeljoffroychandoutis.com/) (filmmaker, Paris) in conversation with Claude (Anthropic), synthesizing production research and AI pipeline work across multiple documentary projects.

Published under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/). Use freely, adapt for your project, credit the source, and share derivatives under the same terms.

Issues, corrections, and war stories welcome.
