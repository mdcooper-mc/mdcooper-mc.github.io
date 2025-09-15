+++
title = "Image Classification"
description = "A GPU‑accelerated framework for organising and enriching large collections of images and videos."
+++

This project was built to bring order to large, mixed‑quality media collections — the kind that blend treasured photos with everyday clutter. It takes a simple list of files and transforms it into a rich, structured dataset that’s easy to search, filter, and work with.

The process starts by checking what’s already known about each file and filling in any missing details. It then creates lightweight, high‑quality thumbnails to speed up processing and reduce strain on storage and networks. These thumbnails are cached so they can be reused without repeatedly loading the original files.

Once the images are prepared, the system uses an AI model to suggest descriptive labels, drawing from a broad vocabulary that blends curated terms with general concepts. It also applies a “junk” filter — a mix of AI scoring and simple rules — to flag content that’s unlikely to be useful, while avoiding false positives on personal or meaningful images.

Beyond classification, the pipeline can identify people and animals in a scene and produce a short, natural‑language description of what’s shown. It works in batches, takes advantage of GPU acceleration when available, and saves progress regularly so it can pick up where it left off.

The architecture is modular, with each stage handling a single task. This makes it easy to adapt, extend, or fine‑tune over time. The result is a fast, reliable way to turn a raw folder of mixed media into an organised, searchable library.

---

## Skills

**Languages & Frameworks**  
Python | PyTorch | Transformers | OpenCV | Pandas | NumPy | NLTK

**AI & Machine Learning**  
Image Labelling | Content Classification | Vocabulary Expansion | Similarity Scoring

**Image & Video Processing**  
Thumbnail Generation & Caching | Metadata Extraction | Video Frame Capture

**Architecture & Design**  
Modular Processing | Cache‑Centric Design | Automated Progress Saving

**Performance & Optimisation**  
GPU Acceleration | Batch Processing | Efficient I/O Handling
