# Repository Review and Testing Plan

## 1) Repository Review

### Current state

The repository currently contains one implementation-facing artifact:

- `FUNCTIONAL_SPECIFICATION.md` (product and pipeline specification).

There is no application source code, test harness, CI workflow, sample dataset, or executable entrypoint in the current tree.

### What the spec already defines clearly

From the functional specification, the intended system behavior is already well-scoped:

- End-to-end workflow from PDF ingestion through dashboard serving.
- Modular architecture boundaries (ingestion, preprocessing, layout/NLP/image processing, persistence, serving).
- Requirements for provenance, confidence scores, deterministic vs. probabilistic boundaries, and reproducibility.

### Key repository gaps to close before implementation testing

1. **No runnable pipeline yet**
   - Add an executable orchestration layer (CLI and/or service).
2. **No test fixtures**
   - Add a versioned fixture set of representative newspaper PDFs and expected outputs.
3. **No acceptance criteria artifacts**
   - Add machine-checkable expected outputs (JSON snapshots) for critical pipeline stages.
4. **No CI checks**
   - Add automated tests that run on every commit.

---

## 2) Testing Plan

The plan is organized to be executable as code is added, while preserving traceability back to the specification.

### A. Test levels

1. **Unit tests (deterministic logic)**
   - File hashing, document ID assignment, metadata normalization.
   - Page count/readability validation.
   - Bounding box geometry transforms.
   - Reading-order and article-assembly rules where deterministic.

2. **Component tests (single stage + fixtures)**
   - OCR decisioning with pages that have good/poor embedded text.
   - Layout detection/segmentation on multi-column pages.
   - Named entity normalization and politician matching using a controlled reference dataset.
   - Image classification interfaces with confidence thresholds.

3. **Integration tests (multi-stage pipeline)**
   - Ingest PDF -> extract pages/text/OCR -> layout blocks -> article objects.
   - Entity/image extraction attached to article/page provenance.
   - Persistence and retrieval consistency checks.

4. **End-to-end acceptance tests**
   - Batch process a fixture corpus.
   - Validate that expected articles/entities/images/metrics are produced.
   - Verify drill-down links from aggregates to page-level evidence.

5. **Non-functional tests**
   - Performance: pages/minute throughput and latency budgets.
   - Reliability: retry behavior and idempotent re-runs.
   - Reproducibility: reprocessing same PDF with same model versions yields same deterministic artifacts.

### B. Core quality gates (recommended CI minimum)

- Gate 1: Deterministic unit tests pass.
- Gate 2: Fixture-based component tests pass.
- Gate 3: One smoke E2E pipeline test on a small PDF batch passes.
- Gate 4: Schema compatibility checks for persisted records.
- Gate 5: Lint/type checks (once language stack exists).

### C. Fixture design for newspaper PDFs

Create a fixture matrix to represent realistic input variability.

1. `digital_text_clean.pdf`
   - Digitally generated PDF with reliable embedded text.
2. `scan_low_quality.pdf`
   - Scanned page(s) requiring OCR, with noise/skew.
3. `mixed_text_scan.pdf`
   - Some pages with usable text layer, others OCR-only.
4. `multi_column_continuation.pdf`
   - Multi-column layout with article continuation across pages.
5. `image_heavy_frontpage.pdf`
   - Multiple photos + captions + advertisements.
6. `ads_dominant_page.pdf`
   - Stress separation of ads vs. editorial content.

For each fixture PDF, store expected outputs per stage:

- ingestion metadata snapshot,
- page-level text/OCR diagnostics,
- layout block summary,
- article reconstruction summary,
- entity/image extraction summary,
- aggregate metrics snapshot.

---

## 3) How to Use a PDF File as Test Input

These instructions are implementation-agnostic so they can be applied once a CLI or API is added.

### Step 1: Prepare the PDF fixture

- Place the PDF in a test-fixture directory, e.g.:
  - `tests/fixtures/pdfs/<fixture_name>.pdf`
- Record fixture metadata in a manifest file (publisher/date/expected characteristics).

Example manifest entry fields:

- `fixture_id`
- `file_name`
- `sha256`
- `expected_page_count`
- `expected_requires_ocr_pages`
- `notes`

### Step 2: Run ingestion/pipeline on the fixture

When implementation exists, run one fixture through the pipeline entrypoint (CLI example pattern):

```bash
newspaper-scanner run \
  --input tests/fixtures/pdfs/mixed_text_scan.pdf \
  --output tests/artifacts/mixed_text_scan \
  --run-id ci-smoke-001
```

(If the project uses an HTTP API instead of CLI, submit the same PDF via multipart upload and capture returned run/document IDs.)

### Step 3: Validate stage-by-stage outputs

For each PDF run, assert the following minimum checks:

1. Ingestion
   - Output hash matches manifest hash.
   - Page count matches expectation.

2. OCR decisioning
   - Pages expected to need OCR are flagged correctly.
   - OCR output includes non-empty token stream and confidence statistics.

3. Layout/segmentation
   - Expected number of article regions is within tolerance.
   - Ads and captions are not merged into article body incorrectly.

4. Reconstruction
   - Continuation pages are linked where expected.
   - Headline/body presence meets threshold checks.

5. NLP/image extraction
   - Expected politician entities are present with confidence above threshold.
   - Expected politician images are detected/classified correctly for fixture cases.

6. Persistence/API
   - Stored records are queryable by document ID.
   - Drill-down references (page number + bounding boxes) resolve.

### Step 4: Compare against golden outputs

- Keep expected outputs in `tests/fixtures/expected/<fixture_id>/`.
- Compare produced JSON outputs to goldens.
- Allow explicit tolerances only for probabilistic fields (confidence, ranking), and keep deterministic fields exact.

### Step 5: Reproducibility check

- Re-run the same PDF fixture with identical model/runtime versions.
- Confirm deterministic artifacts remain identical.
- If probabilistic outputs vary, ensure variation stays within documented tolerance bounds.

---

## 4) Suggested Next Implementation Tasks

1. Add a minimal CLI entrypoint (`run --input <pdf> --output <dir>`).
2. Add fixture folder structure and 2-3 seed PDFs.
3. Add a smoke integration test that processes one fixture and validates ingestion + page decomposition outputs.
4. Add CI workflow to run unit + smoke tests on pull requests.
