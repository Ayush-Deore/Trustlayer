# TrustLayer — AI-Powered Digital Authenticity & Trust

<p align="center">
  <strong>Analyze. Correlate. Verify.</strong>
  <br />
  An AI-powered digital forensics platform designed to identify manipulated media, extract forensic evidence, and establish trust across digital artifacts.
</p>

---

## Overview

Generative AI has fundamentally changed how digital content is created and manipulated. Images can be altered, faces can be replaced, backgrounds can be synthetically generated, and videos can be modified to produce convincing but misleading representations of reality.

Traditional deepfake detection systems typically focus on identifying whether an individual file is real or fake. However, digital authenticity is rarely a single-file problem.

**TrustLayer aims to go beyond isolated deepfake detection.**

It combines computer vision, forensic signal extraction, face recognition, metadata inspection, and cross-artifact analysis to assess the authenticity of digital content.

Rather than relying on a single prediction, TrustLayer treats individual AI models as evidence-producing components within a broader trust assessment system.

Its central question is:

> Do the available digital artifacts provide consistent and credible evidence of the event, identity, and context they claim to represent?

## Core Capabilities

| Capability | Technology | Status |
|---|---|---|
| Image deepfake detection | Vision Transformer (ViT) | Implemented |
| Face-crop deepfake detection | InsightFace + ViT | Implemented |
| Background noise analysis | Laplacian variance | Implemented |
| Face detection and recognition | InsightFace buffalo_l | Implemented |
| Cross-artifact identity matching | FaceRegistry + cosine similarity | Implemented |
| Video frame analysis | FFmpeg + OpenCV | Implemented |
| Video-level manipulation assessment | Frame-level signal aggregation | Implemented |
| EXIF and metadata extraction | Image metadata processing | Implemented |
| Metadata anomaly detection | Heuristic-based checks | Implemented |
| Custom model fine-tuning | PyTorch + Hugging Face | Available |
| Audio forensics | — | Planned |
| Document forensics | — | Planned |
| Full cross-modal evidence fusion | — | Planned |

---

# 1. System Architecture

TrustLayer follows a modular analysis pipeline. Each component extracts a different category of evidence, allowing the system to build a more complete assessment of an artifact.

```text
                 DIGITAL ARTIFACTS
                         |
              +----------+----------+
              |                     |
            IMAGES                VIDEOS
              |                     |
              v                     v
       IMAGE ANALYZER         VIDEO PROCESSOR
              |                     |
       +------+------+       +------+------+
       |      |      |       |             |
      ViT   Face   Noise   Frame Sampling  FFmpeg
       |   Analysis Analysis       |
       |      |      |       Frame Analysis
       +------+------+              |
              |                     |
              +----------+----------+
                         |
                  EVIDENCE EXTRACTION
                         |
          +--------------+--------------+
          |              |              |
     Visual Signals  Identity Data  Metadata
          |              |              |
          +--------------+--------------+
                         |
                  STRUCTURED OUTPUT
                         |
                  FRONTEND VALIDATOR
                         |
                 AUTHENTICITY REVIEW
```

The architecture separates artifact analysis from evidence interpretation. This allows individual detectors to be upgraded or replaced without requiring a complete redesign of the platform.

## 2. Image Analysis Engine

TrustLayer's image analysis engine combines deepfake classification, face-specific analysis, and background-level forensic heuristics.

### 2.1 Vision Transformer Deepfake Detection

The primary image classifier uses the Hugging Face model:

[`dima806/deepfake_vs_real_image_detection`](https://huggingface.co/dima806/deepfake_vs_real_image_detection)

The model analyzes uploaded images and produces classification scores associated with real and manipulated content.

The backend normalizes model outputs into a consistent internal vocabulary:

- `synthetic_visual`
- `authentic_visual`
- `uncertain`

This normalization layer prevents downstream components from depending on the exact label names returned by individual models.

### 2.2 Face-Crop Analysis

Full-image downscaling can destroy subtle facial artifacts and reduce the effectiveness of deepfake detection.

To address this, TrustLayer uses InsightFace to:

1. Detect faces in the uploaded image.
2. Extract bounding boxes for detected faces.
3. Crop individual facial regions.
4. Run deepfake analysis on the isolated face crops.
5. Preserve face-level results alongside full-image analysis.

This creates two distinct analytical perspectives:

- Full-image analysis evaluates the overall visual content.
- Face-crop analysis focuses on potential facial manipulation.

### 2.3 Background Noise Detection

A key component of TrustLayer is its attempt to identify suspicious backgrounds independently of facial authenticity.

The current implementation uses **Laplacian variance** as a measure of image texture and high-frequency detail.

The heuristic evaluates whether background regions exhibit unusually smooth or low-texture characteristics.

It produces normalized signals:

- `synthetic_background`
- `authentic_background`

This is particularly relevant to cases involving:

- AI-generated background replacement.
- Synthetic environments.
- Composited photographs.
- Background regions with unusual texture characteristics.

**Important:** Laplacian variance is a heuristic, not a definitive synthetic-image detector. Natural backgrounds can also be smooth, while generated backgrounds can contain substantial texture. Its output should be interpreted alongside other forensic signals.

### 2.4 Image Resource Management

To maintain stability when processing high-resolution images, the engine automatically resizes large inputs to reduce memory consumption and mitigate out-of-memory errors.

---

# 3. Face Detection & Cross-Artifact Recognition

TrustLayer uses InsightFace with the `buffalo_l` model for face detection and recognition.

### Capabilities

- Face bounding-box extraction.
- Face embedding generation.
- Face similarity comparison.
- Session-scoped identity matching.
- Stable person identifiers across uploaded artifacts.

### FaceRegistry

A session-scoped `FaceRegistry` maintains identity associations across multiple analyzed files.

For example:

```text
image_001.jpg
    |
    +--> Face detected
    +--> Embedding extracted
    +--> person_01 assigned

video_001.mp4
    |
    +--> Same face detected
    +--> Embedding compared
    +--> person_01 matched
```

Cosine similarity is used to compare embeddings and determine whether detected faces are sufficiently similar to an existing session identity.

This allows TrustLayer to associate appearances across images and video frames without treating every detection as an entirely new person.

The registry is session-scoped; identity matching is not a global identity verification service.

---

# 4. Video Analysis Engine

Video analysis extends the image pipeline to temporal media.

Rather than processing every frame, TrustLayer uses intelligent frame sampling to maintain manageable inference times.

### Processing Pipeline

```text
             VIDEO UPLOAD
                  |
                  v
             VIDEO PROBE
                  |
                  v
          DURATION EXTRACTION
                  |
                  v
            FRAME SAMPLING
                  |
             +----+----+
             |         |
           FFmpeg    OpenCV
             |         |
             +----+----+
                  |
                  v
          SAMPLED VIDEO FRAMES
                  |
                  v
           IMAGE ANALYSIS
                  |
                  v
         FRAME-LEVEL SIGNALS
                  |
                  v
         VIDEO-LEVEL ASSESSMENT
```

### Intelligent Frame Sampling

The system uses FFmpeg for frame extraction, with OpenCV as a fallback.

Current safeguards include:

- Maximum of 12 sampled frames per video.
- Duration-adaptive sampling intervals.
- Controlled inference workload.
- Frame-by-frame deepfake analysis.

### Video-Level Confidence

Individual frame predictions are aggregated into a video-level manipulation confidence assessment.

This provides a summarized result without requiring exhaustive frame-by-frame inference.

The current implementation should be understood as sampled-frame analysis rather than continuous temporal forensics.

---

# 5. Metadata & EXIF Forensics

Digital files can contain contextual information that helps establish their origin and processing history.

TrustLayer extracts available metadata and evaluates suspicious indicators.

### Extracted Information

- File size.
- File format.
- Camera make.
- Camera model.
- GPS metadata presence.
- Date and time signatures.
- Available software and editing metadata.

### Anomaly Detection

The system checks for indicators such as:

- Known photo-editing software signatures.
- Suspicious datetime signatures.
- Mismatches between available date/time fields.
- Missing or unusual metadata.

Metadata anomalies are treated as supporting evidence rather than conclusive proof of manipulation.

For example, missing EXIF data does not automatically mean that an image is fake. Social platforms, messaging applications, and image-processing software frequently strip metadata during normal operations.

---

# 6. API & Backend Infrastructure

TrustLayer uses a FastAPI backend to expose analysis functionality.

### Upload Endpoint

`POST /analyze/`

Accepts multiple uploaded image and video files for analysis under a shared session.

Key characteristics:

- Drag-and-drop batch upload support.
- Multiple artifact processing.
- Shared session context.
- Cross-artifact face matching.
- Structured analysis output.

### URL Analysis Endpoint

`POST /analyze/url`

Accepts a public media URL and retrieves the referenced image or video for analysis.

The implementation uses asynchronous HTTP requests through `httpx`.

This allows the backend to process supported public media links without requiring manual local uploads.

URL processing should be restricted to trusted public sources and protected against private-network and server-side request forgery (SSRF) risks.

### Model Initialization & Warm-Up

TrustLayer uses background model initialization during server startup.

Models are loaded and pre-warmed before normal inference requests, reducing the latency associated with loading models on the first analysis request.

### Resource Management

The backend incorporates safeguards to improve inference stability:

- Image resizing.
- Video frame limits.
- Controlled model initialization.
- FFmpeg/OpenCV fallback processing.
- Batch processing within shared sessions.

---

# 7. Custom Model Fine-Tuning

TrustLayer includes a custom training script:

`train_vit.py`

This enables fine-tuning of the baseline Vision Transformer model using organization-specific or domain-specific datasets.

The objective is to improve detection performance on manipulation patterns that are especially relevant to TrustLayer.

### Supported Training Direction

Potential dataset categories include:

| Category | Description |
|---|---|
| Authentic | Unmodified camera photographs |
| Face swap | Images with replaced facial identities |
| Background replacement | Images with generated or replaced backgrounds |
| Object inpainting | Images with objects added or removed |
| Full synthesis | Fully AI-generated images |
| Composite images | Images combining real and synthetic regions |

### Dataset Strategy

TrustLayer's development strategy uses paired real and manipulated samples wherever possible.

```text
       ORIGINAL IMAGE
              |
        Ground Truth
              |
       +------+------+
       |             |
   Authentic     Manipulated
                    |
          +---------+---------+
          |         |         |
       Face Swap  Background  Inpainting
                  Replacement
```

Paired datasets allow controlled evaluation of how manipulation affects individual forensic signals.

The original image is useful as a training and evaluation reference. Production inference, however, should remain capable of analyzing an individual artifact without requiring access to its original.

### Fine-Tuning Objective

The long-term objective is to move beyond binary classification and toward manipulation-aware analysis.

Instead of only identifying whether an image is fake, the engine should increasingly distinguish:

- What appears to have been manipulated.
- Which regions contain suspicious signals.
- Which detectors support the assessment.
- How reliable the available evidence is.

---

# 8. Frontend Validator

TrustLayer currently includes a local dark-mode frontend validator designed to visualize backend analysis results.

### Interface Features

- Dark-mode glassmorphism interface.
- File upload and analysis workflow.
- Live backend health status.
- Model initialization status.
- Manipulation confidence visualizations.
- Person detection badges.
- Expandable metadata accordions.
- Mock-data injection mode.

### Confidence Visualization

Manipulation confidence is displayed using color-coded indicators:

| Color | Interpretation |
|---|---|
| Red | High manipulation signal |
| Yellow | Uncertain or intermediate signal |
| Green | Low manipulation signal |

These colors represent the current analysis output, not a guarantee of authenticity.

The mock-data injection mode allows frontend development and interface testing without repeatedly invoking expensive inference operations.

---

# 9. API Workflow

The current integration flow is:

```text
User
 |
 v
Frontend Validator
 |
 v
FastAPI Backend
 |
 v
POST /analyze/
 |
 v
Artifact Ingestion
 |
 +------------------------+
 |                        |
 v                        v
Image Analysis        Video Analysis
 |                        |
 +------------+-----------+
              |
              v
       Forensic Analysis
              |
     +--------+--------+
     |        |        |
     v        v        v
   ViT     InsightFace Metadata
     |        |        |
     +--------+--------+
              |
              v
      Structured Results
              |
              v
        Frontend UI
```

The backend provides the analysis layer while the frontend focuses on communicating results and supporting investigation.

---

# 10. Development Roadmap

TrustLayer is being developed incrementally, preserving the existing analysis infrastructure while expanding its forensic capabilities.

### Phase 1 — Baseline Forensics

- [x] ViT-based image deepfake detection.
- [x] Face-crop analysis.
- [x] InsightFace face detection.
- [x] Session-scoped FaceRegistry.
- [x] Background noise heuristics.
- [x] Video frame sampling.
- [x] EXIF extraction.
- [x] Metadata anomaly checks.
- [x] FastAPI analysis endpoints.
- [x] Frontend validator.

### Phase 2 — Manipulation-Aware Analysis

- [ ] Region-aware manipulation detection.
- [ ] Dedicated background manipulation detector.
- [ ] Improved synthetic-image detection.
- [ ] Object-level manipulation detection.
- [ ] Improved detection of image composites.
- [ ] Benchmarking across unseen manipulation techniques.
- [ ] Model-specific confidence calibration.

### Phase 3 — Evidence Fusion

- [ ] Unified forensic evidence schema.
- [ ] Detector reliability weighting.
- [ ] Contradiction-aware evidence fusion.
- [ ] Evidence coverage calculation.
- [ ] Explainable manipulation assessments.
- [ ] Dedicated investigation endpoint.

### Phase 4 — Multimodal Forensics

- [ ] Audio authenticity analysis.
- [ ] Speech transcription.
- [ ] Speaker consistency analysis.
- [ ] Document content analysis.
- [ ] Audio-video synchronization checks.
- [ ] Cross-modal entity and timeline consistency.

### Phase 5 — Trust Assessment

- [ ] Evidence graph visualization.
- [ ] Cross-artifact relationship analysis.
- [ ] Coordinated synthetic content detection.
- [ ] Authenticity and manipulation verdicts.
- [ ] Insufficient-evidence handling.
- [ ] Investigation reports.

---

# 11. Design Principles

TrustLayer is guided by several principles that influence both its architecture and its evaluation strategy.

**1. Evidence over isolated predictions**

No single detector should determine the authenticity of an artifact.

**2. Modality independence**

Image, video, audio, and document analyzers should operate independently and expose a common evidence interface.

**3. Explainability**

Every assessment should be accompanied by the signals and observations supporting it.

**4. Uncertainty awareness**

The system should distinguish between low manipulation evidence and insufficient evidence.

**5. Extensibility**

Individual detection models should be replaceable without rewriting the entire analysis pipeline.

**6. Responsible confidence**

Model probabilities are not automatically calibrated probabilities of real-world manipulation. TrustLayer should communicate uncertainty rather than overstate its conclusions.

---

# 12. Technology Stack

| Layer | Technology |
|---|---|
| Backend framework | FastAPI |
| Programming language | Python |
| Deep learning | PyTorch |
| Model ecosystem | Hugging Face Transformers |
| Image classification | Vision Transformer |
| Face detection and recognition | InsightFace |
| Video processing | FFmpeg |
| Computer vision | OpenCV |
| Image forensics | Laplacian variance |
| HTTP requests | HTTPX |
| Frontend | Local Validator UI |
| Model fine-tuning | PyTorch + Hugging Face |

---

# 13. Project Structure

The following represents the logical organization of the current system. Adapt the paths to match the actual repository layout.

```text
TrustLayer/
│
├── backend/
│   ├── main.py
│   ├── models/
│   │   ├── image_detector/
│   │   └── face_analysis/
│   │
│   ├── analyzers/
│   │   ├── image_analyzer.py
│   │   ├── video_analyzer.py
│   │   ├── face_analyzer.py
│   │   └── metadata_analyzer.py
│   │
│   ├── services/
│   │   ├── face_registry.py
│   │   └── model_loader.py
│   │
│   ├── utils/
│   │   ├── preprocessing.py
│   │   └── video_processing.py
│   │
│   └── train_vit.py
│
├── frontend/
│   └── ...
│
├── tests/
│   └── ...
│
├── requirements.txt
└── README.md
```

---

# 14. Getting Started

### Prerequisites

- Python 3.10 or compatible project environment.
- FFmpeg installed and accessible through PATH.
- Git.
- Sufficient RAM and CPU resources for image and video inference.
- GPU acceleration is recommended where supported.

### Clone the Repository

```bash
git clone <YOUR_REPOSITORY_URL>
cd TrustLayer
```

### Create a Virtual Environment

Windows:

```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
```

Linux/macOS:

```bash
python3 -m venv venv
source venv/bin/activate
```

### Install Dependencies

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

Ensure FFmpeg is installed separately and accessible from your terminal.

### Start the Backend

```bash
python -m uvicorn main:app --reload --port 8000
```

If your FastAPI entry point is inside a backend package, run the command from the appropriate directory or adjust the module path.

### Access API Documentation

Once the backend is running:

```text
http://127.0.0.1:8000/docs
```

The interactive Swagger interface can be used to test the available analysis endpoints.

---

# 15. Evaluation Strategy

TrustLayer should be evaluated using more than overall accuracy.

Recommended metrics include:

| Metric | Purpose |
|---|---|
| Precision | How often flagged manipulations are actually manipulated |
| Recall | How many manipulated artifacts are successfully identified |
| F1-score | Balance between precision and recall |
| False positive rate | How often authentic content is incorrectly flagged |
| False negative rate | How often manipulated content is missed |
| Calibration error | Whether confidence scores reflect observed correctness |
| Inference latency | Time required to analyze artifacts |
| Evidence coverage | Proportion of expected evidence successfully extracted |

Evaluation should be stratified by manipulation type, image source, compression level, and unseen generation methods.

This is particularly important because performance on one deepfake dataset does not guarantee generalization to other manipulation techniques.

---

# 16. Limitations

TrustLayer is an evolving forensic analysis system, not an infallible authenticity oracle.

Current limitations include:

- Deepfake detection performance depends on the training distribution of the underlying model.
- The baseline ViT classifier may miss background-only alterations.
- Laplacian variance is a texture heuristic and cannot independently establish synthetic origin.
- Video analysis relies on a limited number of sampled frames.
- EXIF metadata may be absent, stripped, or deliberately modified.
- Face similarity does not establish a person's real-world identity.
- A high detector score does not independently prove that an artifact is manipulated.
- Audio and document forensics are planned capabilities, not part of the currently operational analysis pipeline.

The system should be used as a forensic decision-support tool, with evidence and uncertainty clearly communicated.

---

# 17. Vision

The long-term goal of TrustLayer is to establish a unified evidence-based framework for digital authenticity.

A future investigation should be able to connect multiple artifacts:

```text
IMAGE
  |
  +--> Face identity
  +--> Visual manipulation
  +--> Background consistency
  +--> Metadata
          |
          |
VIDEO ----+
  |
  +--> Frame-level evidence
  +--> Face consistency
  +--> Temporal signals
          |
          |
AUDIO ----+
  |
  +--> Speaker identity
  +--> Synthetic speech indicators
  +--> Transcription
          |
          |
DOCUMENT -+
  |
  +--> Extracted claims
  +--> Entity references
  +--> Timeline information
          |
          v
    EVIDENCE GRAPH
          |
          v
   CROSS-MODAL REASONING
          |
          v
     TRUST ASSESSMENT
          |
          v
  EXPLAINABLE INVESTIGATION
```

The objective is not simply to detect synthetic media. It is to identify inconsistencies, establish relationships, and explain why a collection of digital artifacts should or should not be trusted.

---

## TrustLayer

**From deepfake detection to evidence-based digital trust.**
