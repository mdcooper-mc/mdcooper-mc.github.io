+++
title = "Image Text Extraction Pipeline: VLMs, KV Soft-Reset, and Legal Evidence from Phone Screenshots"
description = "A production pipeline turning hundreds of phone screenshots into court-ready PDFs — VLM tiling, sliding KV window, layer streaming on a 6 GB GPU, and a no-hallucination constraint for legal evidence."
date = "2026-07-23"
math = true
+++

Phone screenshots are not evidence in a useful form. They are unreadable at scale, unsearchable, easy to lose,
and impossible to cross-reference precisely. This project builds a four-stage pipeline that turns hundreds of
raw screenshots — HEIC photos of Teams chats, Outlook emails, Copilot sessions — into clean, structured,
court-ready PDF documents.

The engineering challenges are interesting: the images are tall and narrow; the content scrolls, producing
overlapping captures; the model must never invent; and the entire pipeline runs on a consumer GPU with 6 GB
of true VRAM and a CPU with 32 GB of DDR4.

---

## Pipeline Overview

```
source/*.heic / *.png / *.jpg
       │
       ▼  Stage 1 — vision_extract.py
       │  Qwen2.5-VL-7B (GGUF, llama-cpp) — GPU: 6 GB VRAM, 45 tok/s
       │  Tile each image into overlapping horizontal strips
       │  Late-fusion: encode both strips, decode once
       │  Content-crop: remove status bar and navigation chrome
       │  → output/*.txt  (one .txt per source image)
       │
       ▼  Stage 2 — stitch.py
       │  Qwen2.5-3B (4-bit NF4, HuggingFace)
       │  Sliding window deduplication: 400-word short-term memory
       │  Timestamp gap detection: split documents at 5-minute breaks
       │  Reconstruction principle: best reading per word, never invented
       │  System prompt KV cache pre-computed once, reused per page
       │  → stitched/*.txt  +  _manifest.jsonl
       │
       ▼  Stage 3 — rebuild_llm.py
       │  Qwen2.5-32B-Instruct-Q5_K_M (GGUF, llama-cpp)
       │  KV soft-reset sliding window — one model session per document
       │  Authoritative-source constraint: every output word must exist in input
       │  Layer streaming: ~12 layers GPU VRAM, ~52 layers CPU DDR4
       │  n_batch=32 (not 512) — compute buffer must fit in 6 GB
       │  → structured/*.md  (clean structured Markdown)
       │
       ▼  Stage 4 — export_pdf.py
          WeasyPrint primary / fpdf2 fallback
          Human-readable filename from first document heading
          → pdf/*.pdf  (court-ready output)
```

---

## Stage 1 — VLM Image Tiling

### The pixel budget problem

A phone screenshot is tall and narrow — a full-frame 1080×2340 HEIC renders at approximately 48% of native
resolution inside the VLM's pixel budget (`max_pixels = 1280×28×28 = 1,003,520` tokens by default). At 48%
scale, small-font text in Teams or Outlook becomes unreadable.

The solution is automatic tiling. Each image is split into overlapping horizontal strips, each strip sized to
fill the pixel budget at maximum resolution. A 1080×2340 image tiled into 4 strips renders each strip at
approximately 85% of native — a significant quality improvement for small-body-text recognition.

Strips overlap by 25% so no word is lost at a cut boundary. A message that straddles the cut appears in full
at the bottom of one strip and the top of the next.

```python
def compute_tiles(image_height: int, image_width: int,
                  max_pixels: int = 1280 * 28 * 28,
                  overlap: float = 0.25) -> list[tuple[int, int]]:
    """
    Return (top, bottom) pixel coordinates for each tile.
    Each tile fills max_pixels at full width.
    Adjacent tiles overlap by `overlap` fraction.
    """
    tile_h = int(max_pixels / image_width)
    stride = int(tile_h * (1 - overlap))
    tiles = []
    top = 0
    while top < image_height:
        bottom = min(top + tile_h, image_height)
        tiles.append((top, bottom))
        if bottom == image_height:
            break
        top += stride
    return tiles
```

### Content crop: removing UI chrome

Before tiling, the pipeline removes phone UI chrome. The status bar (time, battery, signal) and navigation
bar at the bottom are detected by pixel-density analysis and cropped. These regions are not content — they
consume the pixel budget without contributing text, and including them can confuse the VLM's spatial
reasoning about message boundaries.

### Late-fusion decoding

Standard tiling produces two separate VLM calls per image — one per strip. Late-fusion instead shows both
strips to the model simultaneously in a single generation call:

> *"Here are two consecutive strips from the same screenshot — transcribe them as one continuous document."*

Since the generation step takes approximately 71% of total per-image processing time, halving the number
of generation calls produces significant throughput improvement. For 1,589 images:

```
Standard:  1,589 images × 2 strips × 2 encode+generate = 6,356 calls
Late-fusion: 1,589 images × 1 encode(both) + 1 generate = 3,178 calls
```

### Speculative decoding (optional)

For predictable, structured text like email headers and chat message formatting, speculative decoding adds
a small draft VLM that proposes 4–8 tokens at once. The main model reviews the draft in a single parallel
verification step and accepts correct tokens without generating them individually.

For structured OCR content, acceptance rate is approximately 70–75% — effectively 2.5× faster token output
with identical quality. The pipeline exposes this as an optional flag (`-DraftModel`) rather than a default,
since the draft model must share vocabulary with the main model.

---

## Stage 2 — Scroll Deduplication and Stitching

### The overlap problem

Screenshots taken while scrolling produce consecutive images where the bottom ~30% of one image appears
again at the top of the next. Simple concatenation produces a document where every paragraph appears 1.3
times. The stitching stage must identify and remove overlapping content without losing any unique text.

### Sliding window deduplication

The pipeline maintains a 400-word sliding window — the "short-term memory" — representing the end of the
document as built so far. For each new page of extracted text, the model receives:

```
[SHORT-TERM MEMORY — last 400 words already written]:
... end of previous content ...

[NEW PAGE]:
... freshly extracted text from current screenshot ...

Task: What new content from [NEW PAGE] should be appended?
Remove any overlap with [SHORT-TERM MEMORY].
Never invent. Prefer the clearest reading of each disputed word.
```

This processes the entire corpus with `O(1)` memory per page rather than `O(n)`.

### Timestamp gap detection

Every screenshot filename encodes its capture timestamp: `20260702_103727.heic` = 10:37:27 on 2 July 2026.
By comparing adjacent timestamps, the pipeline classifies each transition:

```python
def classify_gap(gap_seconds: int) -> str:
    if gap_seconds < 10:   return "continuous_scroll"
    if gap_seconds < 300:  return "short_pause"
    if gap_seconds < 3600: return "session_break"
    return "new_session"
```

At gaps above the configured threshold (default 300 seconds = 5 minutes), the pipeline starts a new document
file. The model receives `[TIME GAP: 16 min — possible context switch]` as an inline annotation so it knows
the next screenshot may be from a different conversation thread.

### The reconstruction principle

When the same text appears in two consecutive strips with different quality, the model picks the clearest
reading of each word from whichever strip has it legible. If a word is unreadable in both strips, it is
**omitted entirely, never invented**. This is the legal evidence constraint: every word in the output must
have been visibly present in a source screenshot.

### System prompt KV cache

The system prompt — approximately 600 words of deduplication rules, reconstruction principles, and
formatting constraints — is pre-computed once and cached as a compressed KV state before processing begins.
This cached state is reused for every page call. Over 1,589 pages, re-reading 600 words each time would
waste approximately 12 minutes of computation.

---

## Stage 3 — KV Soft-Reset Sliding Window

This is the architectural core of the pipeline. The problem is that a complete stitched document may be
12,000–15,000 words — far larger than any reasonable context window. The naive solution resets the model
between chunks, but full resets are expensive: the system prompt and previous output must be re-fed from
scratch on every reset, consuming thousands of tokens of prefill compute.

### KV window progression

The pipeline maintains a continuous model session per document, tracking the KV cache state explicitly:

```
Document start:
  Load model. Feed system prompt (~320 tokens).
  Cache this state as prefix_snapshot.
  Cost: once per document.

Part 1 (~2,000 words of stitched input):
  eval(input_1, ~2,700 tokens)  →  appended to KV
  generate  →  stream to structured/doc_N.md  (one token per ~700ms, flushed immediately)
  KV = [system(320) + input_1(2,700) + output_1(~800)] = ~3,820 tokens

Part 2:
  eval(input_2)  →  appended, no reset
  generate  →  continue streaming to same file
  KV = [system + input_1 + output_1 + input_2 + output_2] = ~7,300 tokens

Part 3  —  overflow detected:
  7,300 + 2,700 (input_3) + 1,024 (minimum output headroom) > 8,192 × 0.88

  SOFT-RESET:
    1. restore prefix_snapshot  (system only, ~320 tokens — instant from cached state)
    2. eval(output_tail)  — last 400 words of clean output already on disk (~540 tokens)
    3. KV = [system(320) + tail(540)] = ~860 tokens
    4. eval(input_3)  →  generate  →  continue streaming

  Cost: ~540 tokens prefill
  vs full reset: ~3,820 tokens prefill  →  7× cheaper
```

### Why feed the model's own output, not the raw input?

After a soft-reset, the model receives its own clean structured output as context — not the raw OCR input.
The model knows exactly what it already wrote. It continues from its last sentence with no repeated content
and no visible seam. A model that instead received raw OCR input after a reset would need to re-deduplicate
and re-structure the same text, introducing inconsistency at every boundary.

### Efficiency gains

| Approach | Prefill cost per reset | For 80-part document |
|---|---|---|
| Full reset (re-feed everything) | ~3,820 tokens | ~305,600 tokens |
| Soft-reset (output tail only) | ~540 tokens | ~43,200 tokens |
| System prefix snapshot | 0 tokens | always free |

Total saving across 80 parts processed: approximately 170,000 tokens of eliminated prefill — roughly
34 hours of compute time at 1.4 tok/s.

### Context window budget (n_ctx = 8192)

```
System prompt:           ~320 tokens   (snapshot cached — free on all resets)
Previous output tail:    ~540 tokens   (only on soft-reset)
Current part input:    ~2,700 tokens   (2,000 words of stitched text)
Message framing:          ~100 tokens
─────────────────────────────────────
Total prompt:           ~3,660 tokens
Output budget:          ~4,500 tokens  (capped: max_new = min(estimate, remaining))
Total:                  ~8,160 tokens  →  n_ctx = 8,192 — exactly fits with headroom
```

### The authoritative-source constraint

The single most important feature of Stage 3 is that the model is never permitted to invent. Every call
includes the original source text labelled as `[AUTHORITATIVE SOURCE]`, and the system prompt explicitly
states:

> *"Every word in your output MUST appear in the AUTHORITATIVE SOURCE. You may not supply any word, name,
> date, or technical term that is not already present. If a word is unclear, reproduce the original unchanged.
> An artefact in an evidence record is preferable to an invented correction."*

This is the legal evidence equivalent of requiring a court reporter to correct a transcript by reference to
the original recording — never by guessing.

### Resumability

Every completed part is moved to `reviewed/processed/` after writing. On restart, completed parts are
detected by filename and skipped — not by index offset, which would be fragile to gaps from empty or
skipped parts. A 47-hour interrupted run resumes cleanly.

---

## Running on Constrained Hardware

The pipeline is designed to run on a consumer machine: RTX 2060 Max-Q, 6 GB GDDR6 VRAM, 32 GB DDR4.

### Layer streaming: fitting a 32B model in 6 GB

A 32B parameter model in Q5_K_M quantisation requires approximately 21.9 GB. The GPU has 6 GB.
`llama-cpp-python` solves this with layer streaming: as many transformer layers as will fit are loaded into
true VRAM; the rest are streamed from CPU DDR4 at inference time.

```python
llm = Llama(
    model_path="models/Qwen2.5-32B-Instruct-Q5_K_M.gguf",
    n_gpu_layers=-1,   # -1 = auto: fill VRAM first, overflow to CPU DDR4
    n_ctx=8192,
    n_batch=32,        # MUST be 32, not the default 512
    type_k=8,          # INT8 K-cache — works on sm_75
    type_v=1,          # FP16 V-cache — INT8 requires Flash Attention (sm_80+)
    verbose=True,
)
```

On RTX 2060 Max-Q (sm_75, 6 GB GDDR6):

```
32B model — 64 layers total:
  ~12 layers  →  GPU VRAM (GDDR6, 192 GB/s)    ← fast path
  ~52 layers  →  CPU DDR4 (~25 GB/s)            ← ~7× slower but acceptable
       0      →  SharedRAM / PCIe (~6 GB/s)     ← NEVER — 4× slower than CPU
```

**Why `n_batch=32` is critical:** The compute buffer is allocated for all layers simultaneously and scales
with batch size. At the default `n_batch=512`, this buffer requires approximately 10 GB — which immediately
exceeds the GPU's 6 GB and forces the entire buffer into SharedRAM (the PCIe-backed system RAM pool that
Windows exposes as GPU memory). The result is a model that appears to load but runs at 0.36 tok/s rather
than 1.4 tok/s. Setting `n_batch=32` keeps the compute buffer to approximately 700 MB.

**Why `type_v=1`, not `type_v=8`:** INT8 V-cache quantisation (`type_v=8`) requires Flash Attention, which
requires compute capability sm_80 or higher. The RTX 2060 Max-Q is sm_75. Attempting `type_v=8` on sm_75
causes a silent context creation failure — the model loads without error but produces garbage output.

**Measured speeds on this hardware:**

| Configuration | Speed |
|---|---|
| SharedRAM (`n_gpu_layers=60`, spills to PCIe) | 0.36 tok/s |
| CPU only (`n_gpu_layers=0`) | 1.13–1.25 tok/s |
| VRAM + CPU (`n_gpu_layers=-1`, ~12 in VRAM) | ~1.4 tok/s |

The ~12 layers in VRAM operate at 192 GB/s vs 25 GB/s — a 7.7× bandwidth advantage per layer. Even with
only ~19% of layers in VRAM, the memory bandwidth improvement produces a 12–25% throughput gain over
CPU-only.

### SharedRAM is not real VRAM

Windows WDDM driver exposes system RAM as a secondary GPU memory pool (shown as "Shared GPU memory" in Task
Manager and as available VRAM in some APIs). This pool runs over PCIe at approximately 6 GB/s — four times
slower than CPU DDR4. Setting `n_gpu_layers` to a value that spills into SharedRAM always produces worse
performance than `n_gpu_layers=0` (CPU-only). The pipeline hard-blocks SharedRAM usage.

---

## Performance Summary

| Configuration | Stage 1 | Stage 2 | Stage 3 | Stage 4 | Total |
|---|---|---|---|---|---|
| Baseline (no optimisations) | ~21 h | ~6 h | — | ~1.5 h | ~28.5 h |
| + Late fusion | ~17.5 h | ~3 h | — | ~1.5 h | ~22 h |
| + Compiled encoder | ~16.5 h | ~3 h | — | ~1.5 h | ~21 h |
| + Speculative decoding | ~9 h | ~3 h | — | ~1.5 h | ~13.5 h |
| + Classification structuring | ~9 h | ~3 h | — | ~0.5 h | ~12.5 h |
| **Full pipeline with Stage 3** | **~9 h** | **~3 h** | **~1 h** | **~0.5 h** | **~13.5 h** |

Total improvement: 28.5 h → 13.5 h — a 2.1× speedup while adding a 32B correction pass.

---

## Model Selection

| Stage | Model | Reason |
|---|---|---|
| Stage 1 — image reading | Qwen2.5-VL-7B Q6_K (GGUF) | Specialist vision model; largest VLM that fits in 6 GB VRAM |
| Stage 2 — stitching | Qwen2.5-3B 4-bit NF4 (HF) | Fast; deduplication task needs minimal language understanding |
| Stage 3 — rebuild | Qwen2.5-32B Q5_K_M (GGUF) | 32B significantly better at contextual correction; GGUF enables layer streaming |
| Stage 4 — PDF | WeasyPrint + fpdf2 fallback | Markdown → PDF with full Unicode; fpdf2 handles edge cases |

**Why not use the 32B model for everything?** Stage 1 requires a vision model — vision capability in
GGUF format exists only up to 7B at usable quality on this hardware. Stage 2 is a deduplication task
where a 3B model is sufficient. Using a 32B model for Stage 2 would add ~3 hours for negligible
quality improvement.

---

## Running the Pipeline

```powershell
# One-time setup
py -3.12 -m venv venv
.\venv\Scripts\Activate.ps1
pip install -r requirements.txt

# Build llama-cpp-python with CUDA (required for Stage 3)
$env:CMAKE_ARGS = "-DGGML_CUDA=on"
$env:FORCE_CMAKE = "1"
pip install llama-cpp-python --upgrade --force-reinstall --no-cache-dir

# Full pipeline run
.\run_pipeline.ps1 -FreshRun

# Resume after interruption (auto-detects completed files)
.\run_pipeline.ps1

# Re-run Stage 3 + 4 only (keep stitched output)
.\run_pipeline.ps1 -RebuildOnly
```

Output appears in `pdf/` with human-readable filenames derived from document headings.
Each stage is fully resumable — interrupted runs pick up from the last completed file.

---

## What This Produces

**Input:** 1,589 phone screenshots spanning 5 documents across 23 work sessions.

**Output:**
```
pdf/
  Communication_Load_and_Work_Management.pdf     (48 KB)
  Repository_Handover_and_Code_Understanding.pdf (52.7 KB)
  12_September_2025_15_32.pdf                    (305.5 KB)
  Performance_Goals_Review.pdf                   (7.5 KB)
  February_2026_Production_Validation.pdf        (2.4 KB)
```

Clean Markdown → typeset PDF with verbatim quotes, timestamps, participant names, and
a provenance header documenting the source screenshot timestamps and filenames.

**[→ v2 Redesign: Event-Driven Architecture](/ai/image-text-extraction-v2/)**

---

*Source: [ImageTextExtraction on GitHub](https://github.com/mdcooper-mc)*

