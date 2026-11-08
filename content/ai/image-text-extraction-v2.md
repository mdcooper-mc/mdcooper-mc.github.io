+++
title = "Image Text Extraction v2: Event-Driven Pipeline Architecture"
description = "A ground-up redesign of the v1 extraction pipeline — dependency injection, event bus, AST-based toolkit, curr+next VLM window, and a fully test-driven build."
date = "2026-07-23"
math = true
+++

The v1 pipeline solved the problem. v2 redesigns the solution as a proper software system — one that is
testable, composable, and maintainable. The core algorithms are unchanged; the architecture around them
is rebuilt from scratch using dependency injection, an event-driven stage model, and a standalone toolkit
library that carries no mandatory runtime dependencies.

**[← v1 Pipeline: Algorithms and Hardware](/ai/image-text-extraction/)**

---

## Why v2?

The v1 pipeline is a set of standalone Python scripts. It works. But it has properties that make it
difficult to extend:

- No dependency injection — components are tightly coupled; testing requires the full stack
- No event bus — stages communicate through shared filesystem state, not interfaces
- No separation between domain logic and infrastructure — file I/O is scattered throughout
- No test coverage — correctness is verified by running the full pipeline and inspecting output
- Configuration is baked into script arguments — no structured config system

v2 addresses each of these. The pipeline logic is unchanged; the delivery mechanism is redesigned.

---

## Toolkit — Zero-Dependency Infrastructure Library

Before building the pipeline, v2 first builds a standalone toolkit library that the pipeline depends on.
The toolkit has **no mandatory third-party dependencies** — the core packages use only the Python standard
library. LLM support is an optional extra.

The toolkit is split into four packages:

### toolkit.events — Publish/Subscribe Event Bus

```python
from toolkit.events import IEventBus, InMemoryEventBus, IEvent, IEventHandler, event_handler

@event_handler("image_transcribed")
class TranscriptionLogger(IEventHandler):
    def handle(self, event: IEvent) -> None:
        print(f"Transcribed: {event.payload()['stem']}")

bus: IEventBus = InMemoryEventBus()
# TranscriptionLogger is automatically subscribed — registered at class definition time
bus.publish(TranscribeCompletedEvent(stem="20260702_103727"))
```

`InMemoryEventBus` is thread-safe, registered as a singleton via `@injectable(IEventBus)`.
`@event_handler(event_type)` registers a handler class with the global handler registry — on construction
of `InMemoryEventBus`, all registered handlers are automatically subscribed.

`NullEventBus` provides a no-op implementation for testing or headless runs.

### toolkit.ioc — Dependency Injection Container

```python
from toolkit.ioc import injectable, inject, bean, configuration, get_container, IContainer

class IVlmService: ...

@injectable(IVlmService)
class VlmService(IVlmService):
    @inject
    def __init__(self, event_bus: IEventBus): ...  # resolved automatically

container: IContainer = get_container()
svc = container.resolve(IVlmService)
```

`@injectable(interface)` registers the decorated class with the process-wide `DIContainer`.
`@inject` patches `__init__` to resolve un-provided constructor parameters by type annotation.
`@bean` and `@prototype` on `@configuration` methods provide singleton and factory bindings.

### toolkit.config — Configuration Injection

```python
from toolkit.config import configure

class VlmService(IVlmService):
    @configure          # injects values from config.yaml at construction
    def __init__(self,
                 model_path: str,
                 mmproj_path: str,
                 n_gpu_layers: int,
                 n_ctx: int,
                 temperature: float,
                 seed: int,
                 max_tokens: int):
        self._model_path = model_path
        self._model = None  # lazy-loaded on first transcribe()
```

`@configure` reads `config.yaml` and injects named constructor parameters by key path. The class never
receives or reads `IConfigurationService` directly — configuration values arrive as plain typed arguments.
This makes the class fully testable without a config file.

```yaml
# resources/dev/config.yaml
pipeline:
  image_transcription:
    model:
      model_path: "../../models/Qwen2.5-VL-7B-Instruct-Q6_K.gguf"
      mmproj_path: "../../models/mmproj-BF16.gguf"
      n_gpu_layers: 100    # 28 layers fit in 6 GB VRAM; clamps automatically
      n_ctx: 4096
      temperature: 0.0
      seed: 42
      max_tokens: 2048
    locations:
      source_dir: "../../source"
      output_dir: "../../output"
```

### toolkit.code — AST Scanner and Documentation Generator

A separate concern: an AST-based Python source scanner and Markdown writer used for generating API
documentation. No third-party dependencies.

```python
from toolkit.code import CodeScanner, MarkdownWriter, DescriptionEnricher, SourceCollector
from toolkit.events import InMemoryEventBus

bus = InMemoryEventBus()
collector = SourceCollector()
bus.subscribe('class_scanned', collector)
bus.subscribe('function_scanned', collector)

scanner = CodeScanner(event_bus=bus, project_root=str(Path.cwd()))
result = scanner.scan('python/src', 'python/tests')

MarkdownWriter().write(result, 'docs/api.md')
```

With optional LLM enrichment:

```python
from toolkit.llm import LlmDescriptionProvider, Summarizer

model = Summarizer("Qwen/Qwen2.5-3B-Instruct")
model.start()
enricher = DescriptionEnricher(LlmDescriptionProvider(model), collector)
result = enricher.enrich(result)
```

The `Summarizer` overrides `summarize_all` with a `ThreadPoolExecutor`-based CPU-GPU pipeline —
tokenisation runs on CPU while the GPU decodes the previous batch, improving GPU utilisation from ~44%
to ~89%.

---

## Pipeline Architecture

The pipeline is structured around four concerns that map cleanly to interfaces:

```
IStage          — a named, runnable unit of work
IWorkflow       — an ordered sequence of stages
IFileService    — all filesystem operations (list, read, write, move)
IEventBus       — publish/subscribe for inter-component communication
```

No stage holds a direct reference to another stage. No stage reads or writes files directly. All
communication is through events. All dependencies are injected at construction time.

```
ApplicationConfiguration.wire(config_path)
       │
       ├─ PluggableWorkflow
       │    ├─ BoundaryDetectionStage
       │    ├─ ImageTranscriptionStage
       │    ├─ DocumentReconstructionStage  ← Stage 4, pending
       │    └─ PdfExportStage               ← Stage 5, pending
       │
       ├─ InMemoryEventBus
       ├─ FileService  (handles ReadDirectory, WriteFile, WriteBinary, MoveToProcessed)
       └─ VlmService  (lazy-loads model on first transcribe())
```

### Stage lifecycle

Each stage follows the same lifecycle contract:

```python
class IStage(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...

    @abstractmethod
    def run(self) -> None: ...
```

The `PluggableWorkflow` iterates stages in order, publishing `StageCompletedEvent` after each one.
The event bus propagates the event to any handler that needs to respond — file cleanup, logging,
progress reporting — without the workflow knowing any of those handlers exist.

---

## Stage 2 — Boundary Detection (Chrome Removal)

Each screenshot contains phone UI chrome: the status bar at the top (time, battery, carrier signal) and
the navigation bar at the bottom (home gesture, back button, task switcher). These must be removed before
the VLM reads the image.

```python
class BoundaryDetectionStage(IStage):
    """
    Publishes CropImageRequestedEvent for each source image.
    ImageCropperService handles the event and writes the cropped result.
    The stage knows nothing about the cropper implementation.
    """
    def run(self) -> None:
        images = self._file_service.list_directory(self._source_dir)
        for image_file in images:
            self._event_bus.publish(
                CropImageRequestedEvent(
                    source_path=image_file.path,
                    output_path=self._output_dir / image_file.stem
                )
            )
```

`ImageCropperService` subscribes to `CropImageRequestedEvent` and performs the crop. It is registered
via `@injectable(IImageCropperService)` and subscribed automatically by `InMemoryEventBus`. The stage
never references `ImageCropperService` directly.

---

## Stage 3 — VLM Transcription: curr + next Window

The key architectural decision in Stage 3 is the curr+next sliding window — a departure from v1's
tiling approach. Each image is always processed alongside the next image in sequence.

### Why curr+next instead of prev+curr?

The critical insight is about which direction resolves seam ambiguity better:

```
Screenshot N-1:  ─────────────────────────────
                 ════════════ overlap ═════════  ← partially covered by UI chrome at bottom
Screenshot N:    ════════════ overlap ═════════  ← unobstructed at top
                 ─────────────────────────────
Screenshot N+1:  ─────────────────────────────
```

Content at the **top** of a screenshot is always unobstructed. Content at the **bottom** may be partially
covered by the phone's keyboard, bottom navigation bar, or input field chrome. The model sees the overlap
region in both images simultaneously and preferentially uses the unobstructed version.

In v1, tiling handles this within a single image. In v2, the VLM sees both images in one call and resolves
seam content correctly without any post-processing deduplication step.

```python
class ImageTranscriptionStage(IStage):
    def run(self) -> None:
        images = sorted(self._file_service.list_directory(self._source_dir))
        for i, current in enumerate(images):
            next_image = images[i + 1] if i + 1 < len(images) else None
            self._event_bus.publish(
                TranscribeImageRequestedEvent(
                    current_image_path=current.path,
                    next_image_path=next_image.path if next_image else None,
                    output_stem=current.stem
                )
            )
        self._transcription_service.unload()  # release VRAM before next stage
```

`ImageTranscriptionService` handles `TranscribeImageRequestedEvent`:

```python
class ImageTranscriptionService(IImageTranscriptionService):
    def handle(self, event: ITranscribeImageRequestedEvent) -> None:
        text = self._vlm_service.transcribe(
            current_path=event.current_image_path,
            next_path=event.next_image_path      # None for the last image
        )
        if not text:
            raise PipelineRuntimeError(f"VLM returned empty for {event.output_stem}")
        self._event_bus.publish(WriteFileRequestedEvent(
            path=self._output_dir / f"{event.output_stem}.txt",
            content=text
        ))
        self._event_bus.publish(MoveToProcessedRequestedEvent(
            source=event.current_image_path
        ))
```

### VlmService — deterministic, lazy-loading

```python
class VlmService(IVlmService):
    @configure
    def __init__(self, model_path, mmproj_path, n_gpu_layers, n_ctx,
                 temperature, seed, max_tokens):
        self._params = locals()
        self._model = None  # loaded on first call to transcribe()

    def transcribe(self, current_path: Path, next_path: Path | None) -> str:
        if self._model is None:
            self._model = Llama(
                model_path=self._params['model_path'],
                chat_handler=Qwen2VLChatHandler(
                    clip_model_path=self._params['mmproj_path']
                ),
                n_gpu_layers=self._params['n_gpu_layers'],
                n_ctx=self._params['n_ctx'],
                seed=self._params['seed'],
                verbose=False,
            )
        prompt = self._build_prompt(current_path, next_path)
        result = self._model.create_chat_completion(
            messages=prompt,
            temperature=self._params['temperature'],
            max_tokens=self._params['max_tokens'],
        )
        return result['choices'][0]['message']['content']
```

All runs are fully deterministic: `temperature=0.0`, fixed `seed`, parameters recorded in `config.yaml`.
The model stays resident in VRAM for the full stage run and is explicitly released before the next stage
by `ImageTranscriptionStage.run()` calling `unload()`.

**Model choice:** `Qwen2.5-VL-7B-Instruct-Q6_K.gguf` (6.25 GB) for the language decoder and
`mmproj-BF16.gguf` (1.35 GB) for the vision projector. The 7B VLM is the largest that fits entirely in
6 GB true VRAM. The BF16 projector is used specifically over F16 — wider numerical range at the same
file size.

---

## Testing Architecture

Every component in v2 is built test-first. The toolkit has 96 tests; the main pipeline has 75 tests.
All 171 tests pass before any stage is declared complete.

The event-driven architecture makes unit testing clean — components are tested in complete isolation by
constructing them with a `NullEventBus` and inspecting published events:

```python
def test_image_transcription_service_publishes_write_event():
    bus = InMemoryEventBus()
    captured = []
    bus.subscribe('write_file_requested', lambda e: captured.append(e))

    vlm = FakeVlmService(returns="Transcribed text")
    service = ImageTranscriptionService(vlm, bus, output_dir=Path("/out"))
    service.handle(TranscribeImageRequestedEvent(
        current_image_path=Path("/src/img.png"),
        next_image_path=None,
        output_stem="img"
    ))

    assert len(captured) == 1
    assert captured[0].payload()['content'] == "Transcribed text"
    assert captured[0].payload()['path'] == Path("/out/img.txt")
```

No filesystem. No model. No config file. Pure interface testing.

The `FakeVlmService` implements `IVlmService` and returns a controlled response — the real `VlmService`
is never instantiated in unit tests.

---

## Current Status

| Stage | Status | Tests |
|---|---|---|
| Stage 0 — Toolkit | ✅ Complete | 96 passing |
| Stage 1 — Pipeline skeleton | ✅ Complete | — |
| Stage 2 — Boundary detection (chrome crop) | ✅ Complete | 75 passing |
| Stage 3 — VLM transcription (curr+next) | ✅ Complete | 75 passing |
| Stage 4 — Document reconstruction | ⏳ Pending plan | — |
| Stage 5 — PDF export | ⏳ Pending | — |

v2 is on hold while v1 is in active use for evidence production. When v2 resumes, Stage 4 implements the
document reconstruction stage — the KV soft-reset sliding window from v1, rebuilt within the v2 event-driven
architecture.

---

## Key Differences: v1 vs v2

| Concern | v1 | v2 |
|---|---|---|
| **Architecture** | Standalone scripts | Event-driven stages with IoC |
| **Testing** | Manual inspection | 171 unit tests, fully isolated |
| **Configuration** | CLI flags | `config.yaml` with `@configure` injection |
| **File I/O** | Scattered `Path.write_text()` calls | `IFileService` via events |
| **VLM window** | Tiling (horizontal strips) | curr+next (consecutive images) |
| **Seam resolution** | Post-processing deduplication | VLM sees both images simultaneously |
| **Dependencies** | Mixed imports throughout | All injected at construction time |
| **Resumability** | File-presence check | Event-driven, stateless per file |

The fundamental algorithms — KV soft-reset for document reconstruction, layer streaming for constrained
hardware, the no-hallucination constraint — carry forward unchanged from v1 into v2's architecture.

**[← v1 Pipeline: Algorithms and Hardware](/ai/image-text-extraction/)**

---

*Source: [ImageTextExtraction on GitHub](https://github.com/mdcooper-mc)*

