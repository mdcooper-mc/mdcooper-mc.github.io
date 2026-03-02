+++
title = "Image Classification & Face Recognition System"
description = "A GPU‑accelerated framework for organising and enriching large collections of images with AI‑powered classification and interactive face recognition."
+++

This project was built to bring order to large, mixed‑quality media collections — the kind that blend treasured photos
with everyday clutter. It takes a simple list of files and transforms it into a rich, structured dataset that's easy to
search, filter, and work with through an interactive 3D visualisation interface.

The process starts by checking what's already known about each file and filling in any missing details. It then creates
lightweight, high‑quality thumbnails to speed up processing and reduce strain on storage and networks. These thumbnails
are cached so they can be reused without repeatedly loading the original files.

Once the images are prepared, the system uses AI models to suggest descriptive labels, drawing from a broad vocabulary
that blends curated terms with general concepts. It also applies a "junk" filter that combines AI scoring with simple
rules to flag content that's unlikely to be useful, while carefully avoiding false positives on personal or meaningful
images.

Beyond classification, the pipeline identifies people using facial recognition, generates short natural‑language
descriptions of scenes, and produces interactive 3D visualisations of face embeddings. The visualisation system allows
real‑time assignment of names to faces, with smooth fog‑like boundaries showing person clusters in 3D space.

The web interface provides an assignment panel where users can click any point in the 3D plot to view the face
thumbnail, see all faces in that image, and quickly assign names through direct input, AI‑powered guessing, or
quick‑select buttons. Changes are saved immediately and reflected across all views.

The architecture is modular, with each stage handling a single task. Processing happens in batches, takes advantage of
GPU acceleration when available, and saves progress regularly so it can pick up where it left off. The result is a
fast, reliable way to turn a raw folder of mixed media into an organised, searchable library with rich metadata and
interactive exploration tools.

---

## The Challenge

Managing large photo collections is a problem that scales poorly with time. What starts as a few hundred vacation
photos becomes thousands of images scattered across folders, mixed with screenshots, memes, and duplicates. Finding a
specific person or event means manually scrolling through thumbnails, hoping you remember which folder and which year.
The chaos compounds when multiple people contribute photos to shared storage, each with their own organisational
system — or lack thereof.

Traditional photo management tools fall into two camps. Consumer applications like Google Photos and Apple Photos lock
your data into proprietary ecosystems with opaque AI processing and privacy concerns. Professional tools like Adobe
Lightroom require manual tagging and organisation, which works beautifully until you face ten thousand untagged images.
Both approaches struggle with network‑attached storage where performance matters and local processing is preferable to
cloud uploads.

This project addresses these challenges by building a GPU‑accelerated pipeline that runs entirely on your own hardware.
It processes images in batches, extracting metadata, detecting faces, generating descriptions, and assigning semantic
labels — all while maintaining a clear separation between raw data and AI‑generated enrichments. The system is designed
to resume gracefully after interruptions, cache aggressively to avoid redundant processing, and present results through
an interactive web interface that makes exploration intuitive.

---

## The Architecture

The system is built around a modular pipeline where each processor handles a single responsibility. The pipeline reads
from a CSV file that tracks the state of every image, runs processors in sequence, and updates the CSV after each block
of images. This design allows the system to resume processing after interruptions without losing progress or
reprocessing completed work.

Five processors run in a carefully ordered sequence, each responsible for a single stage of enrichment. Every
processor receives the same shared DataFrame, reads the rows it needs to work on, applies its model or heuristic,
and writes its result back to the relevant column. This single-responsibility design means any processor can be
replaced, skipped, or extended without touching the others.

### MetaProcessor

MetaProcessor always runs first. Before any AI model touches an image, the system needs to know what it is dealing
with — the file extension, pixel dimensions, file size on disk, video duration if applicable, and any EXIF rotation
flag. MetaProcessor pulls this data from the thumbnail proxy, which caches it so subsequent runs skip files that have
already been measured. The processor also loads any previously detected face metadata from disk, making that data
available downstream without re-running detection. Any row that is missing even one of these fields is selected for
processing, ensuring the pipeline never operates on incomplete metadata.

### FaceProcessor

FaceProcessor uses InsightFace to detect every face in an image and compute a 512-dimensional embedding for each one.
These embeddings capture the geometric structure of a face — the proportions, distances, and angles between landmarks
— in a form that can be compared numerically. Similar faces produce embeddings that are close together in vector
space; different faces sit far apart. The processor assigns each detected face a unique 16-character hexadecimal
identifier and writes the embedding to a separate JSON file keyed by that ID. Detected face metadata, including
bounding boxes, is persisted to a metadata cache so that images processed in a previous session load their results
instantly without re-running the model. The processor also reads from the face database to check which faces already
have names assigned, annotating each row with the current assignment state.

### CaptionProcessor

CaptionProcessor generates a natural-language description of each image using Microsoft's GIT model, pre-trained on
the COCO dataset. The model produces a concise, grammatically correct sentence describing what is depicted — a street
scene becomes "a group of people walking down a busy street at night", a family photograph becomes "a woman and two
children sitting on a couch". These captions are not labels; they are free-form sentences that capture spatial
relationships and context in a way that individual tags cannot. The processor skips any row already marked as junk,
since generating descriptions for low-quality content wastes GPU cycles and produces results that will never be used.

### LabelsProcessor

LabelsProcessor assigns semantic labels using OpenCLIP, a vision-language model trained on LAION-2B. CLIP projects
both images and text into a shared vector space, meaning the embedding for a photograph of a dog sits close to the
embedding for the word "dog". The processor computes an image embedding and scores it against a vocabulary of curated
terms, selecting the top fifteen by similarity. The vocabulary started at fifty thousand words and has been refined
through iterative processing down to a stable subset that produces consistent, meaningful results. The processor
reuses image features already computed earlier in the session from the model cache, so the forward pass through CLIP
happens only once per image regardless of how many downstream tasks consume its output. Like the caption processor, it
skips images already flagged as junk.

### JunkProcessor

JunkProcessor runs last so it has access to all prior outputs. It assigns every image a junk probability between zero
and one hundred based on a combination of semantic scoring and metadata signals. Images containing a known, named face
receive an immediate score of zero — if a person has been identified in a photo, the photo is by definition not junk.
Images with safe-content labels such as drawings or artwork are similarly protected. For everything else, the
processor blends a CLIP semantic score against a vocabulary of junk-associated terms with metadata-based signals:
small PNG files under 500KB that are often screenshots or icons, images with a pixel area under 10,000 that are too
small to contain useful content, and narrow-aspect images under 300KB that are likely banners or UI elements. Caption
text is also scanned for keywords associated with low-quality content. Each signal adds a weighted boost to the base
score, capped to prevent any single signal from dominating. This layered approach catches the vast majority of
unwanted content while carefully avoiding false positives on personal or meaningful images.

---

## Model Selection and Memory Management

The system is optimised for an NVIDIA RTX 2060 Max‑Q with 6GB of VRAM. This hardware constraint drove careful model
selection and a sequential loading strategy that prevents VRAM exhaustion. OpenCLIP trained on LAION‑2B provides
semantic understanding for labelling. Microsoft GIT trained on COCO generates natural captions. InsightFace handles
facial recognition. These models were chosen for their balance of accuracy and memory footprint, with approximately
2.7GB active during processing against a 6GB budget.

Models are loaded one at a time, used for their batch, then unloaded before the next model loads. This sequential
approach trades some throughput for reliability, ensuring the system never runs out of VRAM during processing. The
thumbnail cache and model feature cache further reduce memory pressure by avoiding redundant loads and forward passes.

---

## The Visualisation Interface

The web interface presents a three‑dimensional scatter plot where each point represents a face embedding projected into
3D space using t‑SNE dimensionality reduction. Points are colour‑coded by person, with the plot using Plotly's WebGL
backend for smooth interaction even with thousands of points.

Person clusters are wrapped in smooth, fog‑like boundaries computed using alphahull surface reconstruction. These
boundaries have 20% opacity, allowing users to see intersections and overlapping clusters while maintaining visual
clarity. Lighting effects including ambient, diffuse, and Fresnel highlights give the surfaces depth and make the 3D
structure easier to parse. Individual boundaries can be toggled per person and are hidden by default, surfacing only
when the user wants to understand cluster relationships.

The legend sits on the left side of the interface, presenting each person's name with a colour indicator and three
toggle columns — one for assigned faces, one for unsure faces, and one for the cluster boundary. Global toggles at the
top affect all people at once. System categories like Junk, NoOne, and Unassigned appear below a separator line,
visually distinguishing them from named individuals.

Clicking any point in the plot opens the assignment panel in a two‑column layout. The left column displays a large
thumbnail of the full image, sized to fill the available space for easy viewing. The right column holds all assignment
controls. Colour‑coded chips at the top represent every face detected in that image, each chip matching the colour of
that person in the plot. Clicking a chip switches the thumbnail view to that face and updates the controls to operate
on the selected face.

A text input accepts name entry with autocomplete suggestions drawn from known people. A Guess button triggers an
AI‑powered name suggestion based on embedding similarity, finding the nearest named embedding in the database and
proposing the most likely match. Action buttons handle the most common operations: Assign confirms the current name,
Mark Junk flags the face as non‑person content, Mark NoOne indicates the detection was a false positive, and Clear
Assignment returns the face to the unassigned state.

A row of quick‑assign buttons at the bottom of the panel displays the most frequently assigned people, allowing
single‑click assignment without requiring text entry. Changes take effect immediately, the plot updates in place, and
a toast notification slides in from the right to confirm the action without interrupting the workflow. Each toast is
colour‑coded by type and fades out automatically, though users can dismiss them manually.

---

## Data Architecture and Storage

The system uses a repository pattern with a clean interface boundary between business logic and data access. The
DataStoreIO interface defines operations for reading and writing face data at the level of intent rather than
implementation. The JsonFileIO implementation provides thread‑safe access to JSON files using locks, ensuring
consistency when multiple requests arrive simultaneously.

Three JSON files store persistent data. The face database maps person names to lists of face IDs, distinguishing
between assigned and unsure faces. The embeddings file stores 512‑dimensional vectors indexed by face ID. The metadata
file maps image paths to face detection results including bounding boxes and face IDs. This separation allows
embeddings to be computed once and reused, while assignments can change freely without touching the embedding data.

The CSV file tracks pipeline state with one row per image and columns for every processor's output. Processors read
the current state, compute missing values, and write updates back, making the pipeline stateless and resumable. The
file serves as both a checkpoint system and a queryable dataset.

This architecture prepares for future migration to SQL. The business logic layer operates on domain objects returned
by repository interfaces, so swapping JsonFileIO for SqliteIO or PostgresIO requires no changes to the visualisation
or processing code. The interface isolates the rest of the system from storage concerns entirely.

---

## Code Quality and Standards

The codebase follows strict quality standards throughout. Functions are limited to fifteen lines to encourage single
responsibility and readability. Classes contain no more than ten methods, and files cap at two hundred lines. When
these limits are exceeded, the code is split by domain rather than by technical type, resulting in a domain‑driven
folder structure where related concerns live together.

The Python code uses full imports without sys.path manipulation, leverages generators and comprehensions for
efficiency, and employs logging instead of print statements for observable behaviour. The JavaScript code uses ES6
features including const and let declarations, async/await for asynchronous operations, and arrow functions. Inline
styles were eliminated in favour of CSS classes, and magic strings were extracted into constants.

The frontend uses no framework dependencies, relying on vanilla JavaScript with modern browser APIs. CSS Grid and
Flexbox handle layouts, with a design token system providing consistent spacing, colours, typography, and border radii
across the interface. The mobile‑first responsive design adapts gracefully across screen sizes.

Error handling follows a structured approach where exceptions are never silently swallowed. Errors are logged
meaningfully, and the system fails fast on invalid input rather than attempting heroic recovery. Type hints appear
throughout the Python code, keeping intent explicit and tooling effective.

---

## Performance and Caching

Performance optimisation happens at multiple layers. Thumbnails are generated once and cached to disk, with the cache
checked before loading any original image. Model features computed during a session stay in memory so subsequent
requests for the same image reuse existing embeddings without a second forward pass. Face metadata is persisted to
JSON after detection, allowing processed images to skip the detection step entirely on future runs.

The pipeline processes images in blocks of 512, updating the CSV after each block. This block size balances memory
usage against checkpoint frequency — larger blocks reduce I/O overhead but increase the work lost if processing is
interrupted, while smaller blocks checkpoint more frequently at the cost of write throughput.

GPU acceleration is used wherever the models support it, with batch sizes tuned to maximise throughput without
exceeding VRAM limits. The web interface uses Plotly's WebGL backend for hardware‑accelerated rendering, keeping the
3D scatter plot smooth and responsive even with thousands of points. The CSV table implements server‑side pagination
to avoid loading the entire dataset into the browser at once.

---

## Future Directions

The immediate next step is migrating from JSON to a SQL database, likely SQLite for single‑user scenarios or
PostgreSQL for multi‑user deployments. Modern SQL databases support vector and array column types, enabling efficient
storage of embeddings and label lists, and transactions ensure that assignments are atomic even under concurrent
access.

The guessing algorithm can be enhanced by leveraging cross‑image context. Since the same person cannot appear twice
in one image except in reflections, faces in an image where a person has already been assigned can be eliminated as
candidates for that person. This constraint propagates across the collection, improving accuracy especially in family
photo scenarios where the same group appears across many images.

An event system would decouple components that currently communicate through function calls and shared state.
Assignment events could trigger plot updates, database changes could notify UI components, and cache invalidation
could happen through an event bus. This pattern would simplify testing and make the system more maintainable as
complexity grows. Additional processors could extend the pipeline without modifying existing stages — video frame
extraction, perceptual duplicate detection, and GPS‑based clustering are all natural extensions of the current design.

---

## Conclusion

This project demonstrates that sophisticated AI‑powered media management doesn't require cloud services or proprietary
ecosystems. With careful architecture, thoughtful model selection, and attention to code quality, it's possible to
build systems that are fast, maintainable, and entirely under your control. The combination of GPU acceleration,
aggressive caching, modular design, and an interactive visualisation interface creates a tool that's both powerful and
genuinely pleasant to use.

The skills developed here span the full stack, from PyTorch model integration and CUDA optimisation to responsive web
design and event‑driven JavaScript. The emphasis on clean architecture, interface segregation, and domain‑driven
design ensures the codebase remains maintainable as requirements evolve. The result is a system that brings order to
chaos, turning a folder of mixed media into an organised, searchable library with rich metadata and intuitive
exploration tools.

---

## Skills

**Languages & Frameworks**  
Python | PyTorch | Transformers | OpenCV | Pandas | NumPy | NLTK | JavaScript (ES6+) | HTML5 | CSS3

**AI & Machine Learning**  
Face Recognition | Image Labelling | Scene Captioning | Content Classification | Embedding Similarity | Vocabulary Expansion

**Image & Video Processing**  
Thumbnail Generation & Caching | Metadata Extraction | Face Detection & Bounding Boxes | Video Frame Capture

**Data Architecture**  
Repository Pattern | Interface Segregation | Domain‑Driven Design | Cache‑First Design | JSON & SQL Datastores

**Web & Visualisation**  
Interactive 3D Visualisation | WebGL Rendering | Event‑Driven UI | Responsive Layout | Toast Notifications

**Architecture & Design**  
Modular Processing | Strategy Pattern | Dependency Injection | Separation of Concerns | Automated Progress Saving

**Performance & Optimisation**  
GPU Acceleration | Batch Processing | Sequential Memory Management | Efficient I/O Handling
