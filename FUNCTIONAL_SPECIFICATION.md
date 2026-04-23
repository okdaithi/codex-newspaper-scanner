# Functional Specification: Newspaper PDF Analysis Application

## 1. System Overview

### 1.1 Purpose

The system ingests newspaper PDFs, extracts structured article content and politician photographs, identifies politician entities in text and images, computes analytical metrics, and serves results through a web dashboard.

### 1.2 Primary outputs

1. Article records with text, metadata, layout coordinates, and confidence scores.
2. Detected images classified as politician-related or non-politician-related.
3. Named entities extracted from articles, including politician names and inferred affiliations where supported.
4. Aggregated analytics for dashboard display.
5. Traceable, versioned, reproducible processing outputs.

### 1.3 Operating mode

Initial mode: batch processing of PDF collections.
Extension mode: near-real-time ingestion with the same pipeline contracts.

### 1.4 Core design principles

1. Separate deterministic processing from probabilistic model outputs.
2. Preserve every intermediate artifact required for audit and reproducibility.
3. Treat each PDF page as a structured document image plus optional embedded text layer.
4. Use layout-aware processing rather than plain text extraction alone.
5. Maintain explicit confidence, provenance, and versioning for every extracted record.

### 1.5 Working assumptions

1. PDFs may be scanned, digitally generated, or mixed.
2. OCR will be required for at least some documents.
3. No reliable pre-existing metadata is available in PDFs.
4. Politician recognition will rely on a reference dataset and/or external model.
5. Newspaper layouts will vary substantially across titles and dates.
6. The system must support local deployment and cloud deployment with equivalent logical behavior.

---

## 2. End-to-End Workflow (Step-by-Step Pipeline)

### 2.1 Pipeline stages

1. PDF ingestion.
2. File normalization and integrity validation.
3. Page rendering and text-layer extraction.
4. OCR on pages lacking sufficient embedded text.
5. Layout detection and page segmentation.
6. Content classification into article blocks, captions, photos, ads, and non-content regions.
7. Article assembly across columns and pages.
8. Named entity extraction from article text.
9. Image detection and politician classification.
10. Politician identity resolution against reference data.
11. Persistence of structured records and lineage artifacts.
12. Analytics aggregation.
13. API serving.
14. Dashboard presentation.

### 2.2 Processing flow

#### Step 1: Ingestion

Input: PDF file or batch of PDF files.
Actions:

* Compute file hash.
* Assign immutable document ID.
* Capture source metadata: filename, source path, ingestion timestamp, batch ID, publisher/date if known.
* Validate file opens and page count is readable.
* Store original binary unchanged.

Output:

* Ingestion record.
* Raw PDF artifact reference.

#### Step 2: Page decomposition

Actions:

* Split PDF into pages.
* Extract embedded text layer where present.
* Render each page to normalized raster image.
* Preserve page order and geometry.

Output:

* Page-level image artifact.
* Page-level extracted text artifact.
* Page metadata.

#### Step 3: OCR decisioning

Actions:

* Estimate text-layer quality per page.
* Determine whether OCR is needed.
* If OCR required, run OCR on page image.
* Generate word-level bounding boxes and confidence values.
* Merge OCR text with embedded text only under explicit rules.

Output:

* OCR text layer.
* OCR confidence map.
* Word/line bounding box data.

#### Step 4: Layout analysis

Actions:

* Detect reading order, columns, headers, footers, article bodies, image regions, caption regions, ad regions, whitespace separators.
* Identify page-level structure.
* Produce block graph for each page.

Output:

* Structured page layout blocks.
* Region labels and confidence scores.

#### Step 5: Segmentation

Actions:

* Group contiguous blocks into candidate articles.
* Associate captions with nearby images.
* Separate advertisements from editorial content.
* Split or merge ambiguous regions using layout and text continuity rules.

Output:

* Candidate article objects.
* Candidate image objects.
* Ad objects.
* Caption objects.

#### Step 6: Article reconstruction

Actions:

* Rebuild article text from ordered blocks.
* Resolve multi-page continuation.
* Preserve page and column coordinates per article segment.
* Attach headline, byline, deck/subheadline, body, and continuation markers where detected.

Output:

* Article records with structured sections and provenance.

#### Step 7: NLP entity extraction

Actions:

* Run entity recognition on article text.
* Detect person names, titles, political roles, parties, ministries, electorates, institutions.
* Normalize names.
* Infer affiliation only when evidence is sufficiently strong.
* Link ambiguous mentions to candidate entities.

Output:

* Entity mentions.
* Normalized entities.
* Candidate politician matches.
* Affiliation candidates and confidence.

#### Step 8: Image processing

Actions:

* Detect photographs, portraits, and face regions.
* Determine whether image depicts a politician, likely politician, or non-politician.
* Match detected faces or image context to known politician reference set if available.
* Use caption and surrounding text as auxiliary signals.

Output:

* Image records.
* Face detections.
* Politician image classifications.
* Identity resolution candidates.

#### Step 9: Aggregation

Actions:

* Compute issue-level, date-level, newspaper-level, and politician-level metrics.
* Generate deduplicated counts and trend series.
* Calculate confidence-weighted and raw counts separately.

Output:

* Analytics tables and dashboard-ready aggregates.

#### Step 10: Serve results

Actions:

* Expose document, article, entity, image, and metrics endpoints.
* Support filters by date, newspaper, politician, article type, confidence, and page.
* Provide drill-down from summary metrics to source page images and bounding boxes.

---

## 3. Component Architecture (Modular Breakdown)

### 3.1 Ingestion Layer

Responsibilities:

* Accept PDFs from upload, file watch, API, or batch import.
* Verify file integrity.
* Assign canonical IDs.
* Manage source metadata and batch metadata.

Inputs:

* PDF binary.
* Optional source metadata.

Outputs:

* Document registry entry.
* Immutable raw artifact.

Deterministic/probabilistic:

* Fully deterministic.

### 3.2 Preprocessing Layer

Responsibilities:

* Render pages.
* Extract embedded text.
* OCR pages as required.
* Normalize image quality where safe.
* Preserve page geometry.

Inputs:

* Raw PDF.
* Document registry record.

Outputs:

* Page images.
* Page text.
* OCR outputs.
* Quality metrics.

Deterministic/probabilistic:

* Rendering deterministic.
* OCR probabilistic.
* Quality decision rules deterministic.

### 3.3 Layout Detection Layer

Responsibilities:

* Detect structural page blocks.
* Classify regions by type.
* Estimate reading order.

Inputs:

* Page image.
* OCR text with bounding boxes.
* Embedded text layer.

Outputs:

* Block graph.
* Block labels.
* Block confidence scores.

Deterministic/probabilistic:

* Model outputs probabilistic.
* Region reconciliation rules deterministic.

### 3.4 Content Segmentation Layer

Responsibilities:

* Separate editorial content from ads, captions, photos, and other regions.
* Assemble article units from blocks.
* Track continuations across pages.

Inputs:

* Labeled page blocks.
* Page metadata.

Outputs:

* Candidate article objects.
* Article segment map.
* Ad and caption records.

Deterministic/probabilistic:

* Hybrid. Block grouping rules deterministic with probabilistic inputs.

### 3.5 NLP Entity Extraction Layer

Responsibilities:

* Identify people, organizations, roles, political affiliations, and place references.
* Normalize and deduplicate names.
* Assign politician candidate scores.

Inputs:

* Article text.
* Caption text.
* Metadata and reference datasets.

Outputs:

* Entity mentions.
* Entity canonicalization records.
* Politician candidates.

Deterministic/probabilistic:

* Entity extraction probabilistic.
* Canonical linking may be rule-based plus model-assisted.

### 3.6 Image Analysis Layer

Responsibilities:

* Detect photos and faces.
* Classify images as politician-related or not.
* Link image to politician identity where possible.

Inputs:

* Page image.
* Image regions.
* Caption/context text.
* Politician reference dataset.

Outputs:

* Image objects.
* Face objects.
* Classification outputs.
* Identity matches.

Deterministic/probabilistic:

* Detection/classification probabilistic.

### 3.7 Storage Layer

Responsibilities:

* Persist raw, intermediate, and final artifacts.
* Maintain versioning and lineage.
* Support retrieval by document, page, article, entity, and image.

Inputs:

* All pipeline outputs.

Outputs:

* Queryable records.
* Immutable artifact store.

### 3.8 Analytics Layer

Responsibilities:

* Aggregate metrics.
* Compute trends and confidence-weighted statistics.
* Provide export datasets.

Inputs:

* Structured article and image records.
* Entity and identity outputs.

Outputs:

* Metric tables.
* Time-series summaries.
* Dashboard data models.

### 3.9 API Layer

Responsibilities:

* Serve search, retrieval, analytics, and drill-down endpoints.
* Enforce access control and pagination.
* Expose provenance and confidence fields.

### 3.10 Frontend Dashboard

Responsibilities:

* Present issue-level and aggregated analytics.
* Support drill-down to page and region level.
* Enable manual review and annotation if required.
* Provide filters and comparative views.

---

## 4. Data Model & Schemas (Tables / JSON Structures)

## 4.1 Canonical identifiers

All records must carry immutable IDs:

* document_id
* page_id
* block_id
* article_id
* image_id
* face_id
* entity_id
* mention_id
* run_id
* model_version_id
* dataset_version_id

## 4.2 Document record

```json
{
  "document_id": "doc_...",
  "source_uri": "string",
  "file_hash": "sha256...",
  "filename": "string",
  "ingested_at": "timestamp",
  "publisher": "string|null",
  "publication_date": "date|null",
  "edition": "string|null",
  "page_count": 0,
  "file_type": "pdf",
  "processing_status": "queued|processing|complete|failed|partial",
  "current_run_id": "run_..."
}
```

## 4.3 Page record

```json
{
  "page_id": "page_...",
  "document_id": "doc_...",
  "page_number": 1,
  "render_width_px": 0,
  "render_height_px": 0,
  "rotation_deg": 0,
  "embedded_text_present": true,
  "ocr_required": true,
  "page_quality_score": 0.0,
  "skew_angle_deg": 0.0,
  "language_candidates": ["en"],
  "layout_version": "v...",
  "ocr_version": "v..."
}
```

## 4.4 Block record

```json
{
  "block_id": "block_...",
  "page_id": "page_...",
  "block_type": "headline|body|caption|photo|ad|footer|header|sidebar|unknown",
  "bbox": {"x": 0, "y": 0, "w": 0, "h": 0},
  "reading_order": 0,
  "text": "string|null",
  "confidence": 0.0,
  "source": "embedded_text|ocr|layout_model|rule",
  "linked_image_id": "img_...|null"
}
```

## 4.5 Article record

```json
{
  "article_id": "art_...",
  "document_id": "doc_...",
  "page_ids": ["page_1", "page_2"],
  "headline": "string|null",
  "subheadline": "string|null",
  "byline": "string|null",
  "body_text": "string",
  "language": "en|null",
  "article_type": "news|opinion|feature|sports|business|politics|other|unknown",
  "start_page": 1,
  "end_page": 2,
  "segment_refs": ["block_1", "block_2"],
  "confidence": 0.0,
  "version": "v...",
  "created_at": "timestamp"
}
```

## 4.6 Image record

```json
{
  "image_id": "img_...",
  "page_id": "page_...",
  "bbox": {"x": 0, "y": 0, "w": 0, "h": 0},
  "image_type": "photo|illustration|logo|ad_image|unknown",
  "caption_text": "string|null",
  "face_count": 0,
  "politician_probability": 0.0,
  "classification": "politician|non_politician|uncertain",
  "identity_candidate_ids": ["person_..."],
  "confidence": 0.0
}
```

## 4.7 Face record

```json
{
  "face_id": "face_...",
  "image_id": "img_...",
  "bbox": {"x": 0, "y": 0, "w": 0, "h": 0},
  "embedding_id": "emb_...",
  "quality_score": 0.0,
  "identity_candidate_ids": ["person_..."],
  "confidence": 0.0
}
```

## 4.8 Entity mention record

```json
{
  "mention_id": "men_...",
  "article_id": "art_...",
  "source_text": "string",
  "entity_type": "person|organization|party|role|location|event",
  "start_char": 0,
  "end_char": 0,
  "normalized_text": "string",
  "canonical_entity_id": "ent_...|null",
  "confidence": 0.0
}
```

## 4.9 Canonical person entity

```json
{
  "entity_id": "ent_...",
  "entity_type": "person",
  "preferred_name": "string",
  "aliases": ["string"],
  "affiliations": [
    {
      "affiliation_type": "party|office|institution",
      "name": "string",
      "start_date": "date|null",
      "end_date": "date|null",
      "confidence": 0.0
    }
  ],
  "reference_source_ids": ["ref_..."],
  "is_politician": true
}
```

## 4.10 Processing run record

```json
{
  "run_id": "run_...",
  "document_id": "doc_...",
  "pipeline_version": "v...",
  "model_versions": {
    "ocr": "v...",
    "layout": "v...",
    "ner": "v...",
    "vision": "v..."
  },
  "dataset_versions": {
    "politician_reference": "v..."
  },
  "started_at": "timestamp",
  "completed_at": "timestamp|null",
  "status": "queued|running|complete|failed|partial",
  "error_summary": "string|null"
}
```

## 4.11 Analytics table examples

### Issue-level metrics

* document_id
* publication_date
* total_articles
* total_images
* politician_images
* unique_politicians_mentioned
* article_count_politics_topic
* confidence_weighted_politician_mentions
* ad_area_ratio
* OCR_error_rate_estimate

### Politician-level metrics

* entity_id
* name
* mention_count
* article_count
* image_count
* confidence_weighted_presence
* date_series
* newspaper_series

---

## 5. PDF Processing Specification

### 5.1 Supported input classes

1. Digitally generated PDFs with selectable text.
2. Scanned PDFs with no embedded text.
3. Mixed PDFs with partial text layers and scanned pages.
4. Rotated or skewed scans.
5. Low-resolution source files.

### 5.2 PDF normalization rules

1. Store original file unchanged.
2. Never overwrite raw source.
3. Normalize page dimensions only in derived artifacts.
4. Preserve original page order.
5. Preserve page rotation metadata.
6. Do not flatten distinct processing paths into one artifact without lineage.

### 5.3 Page quality assessment

Each page receives a quality profile:

* resolution estimate
* skew estimate
* blur estimate
* contrast estimate
* OCR confidence estimate
* text-layer completeness estimate
* layout complexity estimate

### 5.4 OCR selection logic

Use OCR when:

1. No embedded text layer exists.
2. Embedded text coverage is below threshold.
3. Embedded text extraction confidence is low.
4. Layout inference needs OCR bounding boxes.

Do not OCR when:

1. Embedded text layer is high quality and complete.
2. OCR confidence is predicted to be materially worse than embedded extraction.

### 5.5 OCR output requirements

OCR output must include:

* word text
* bounding boxes
* line grouping
* confidence per word
* page-level confidence summary
* optional language detection output

### 5.6 OCR error handling

1. Retain original OCR confidence values.
2. Flag low-confidence text spans.
3. Preserve alternative readings when available.
4. Avoid silent normalization of ambiguous words.
5. Route low-confidence pages to review queue if configured.

### 5.7 Reproducibility requirements

Every OCR output must record:

* OCR engine version
* model weights version
* preprocessing parameters
* page render settings
* language model set
* runtime configuration

---

## 6. Article Detection Logic

### 6.1 Definition of an article

An article is a coherent editorial text unit with a single principal story, possibly spanning multiple columns and pages.

### 6.2 Article boundaries

Detection must use:

* headline detection
* text continuation cues
* column structure
* font and style changes
* article separators
* page and column adjacency
* semantic continuity

### 6.3 Distinguishing content types

The system must classify page regions as:

* article body
* headline
* subheadline/deck
* byline
* caption
* photo
* advertisement
* sidebar
* table
* footer/header
* unknown

### 6.4 Deterministic rules

Use deterministic logic for:

1. Linking caption text to nearest photo under spatial and textual constraints.
2. Treating repeated mastheads, page numbers, and headers as non-article content.
3. Splitting blocks at strong separators such as rules, gutters, and whitespace gaps.
4. Excluding ad regions based on layout and lexical cues.

### 6.5 Probabilistic components

Use model-based classification for:

1. Headline detection in noisy layouts.
2. Article continuation detection.
3. Ad vs editorial discrimination in ambiguous blocks.
4. Block type classification when rules are insufficient.

### 6.6 Multi-page article assembly

Article continuation logic must consider:

1. Same headline or continuation markers.
2. Semantic similarity of adjacent text blocks.
3. Layout alignment across pages.
4. End-of-column continuation cues.
5. Page breaks with hyphenation and carry-over lines.

### 6.7 Edge cases

1. Single article spanning multiple columns and pages.
2. Multiple short articles packed on one page.
3. Advertorial content disguised as editorial layout.
4. Repeated headline fragments on continuation pages.
5. Cut-off OCR at page edges.
6. Inverted or vertically compressed scans.

### 6.8 Article confidence

Each article receives a confidence score based on:

* text extraction quality
* layout certainty
* continuity certainty
* ad separation certainty
* entity coherence

---

## 7. Image Detection & Classification Specification

### 7.1 Image detection scope

Detect:

* photographs
* embedded images
* portrait crops
* photo montages
* logos and non-photo graphics
* ads containing images
* image captions

### 7.2 Photo versus non-photo classification

Classify image regions as:

* photograph
* illustration
* logo
* decorative graphic
* ad image
* unknown

### 7.3 Face detection requirement

For images classified as photographs:

* detect faces where present
* estimate face quality
* assign embeddings for identity matching
* preserve all detected face regions even when identity is uncertain

### 7.4 Politician-related image classification

Classify each image into:

* politician
* non-politician
* uncertain

Signals used:

1. Face recognition against reference dataset.
2. Caption text.
3. Nearby article text.
4. Contextual political role cues.
5. Newspaper section/topic.
6. Known appearance patterns if reference data supports them.

### 7.5 False positive control

The system must actively suppress:

* generic crowd photos
* unrelated public figures
* sports figures
* celebrity images
* stock photos
* editor portraits
* images of politicians in non-political contexts unless the reference policy permits inclusion

### 7.6 Low-quality image handling

For blurred or partial images:

* retain as candidate records
* lower confidence
* avoid forced identity assignment
* preserve bounding boxes and context

### 7.7 Image provenance

Each image record must store:

* source page
* surrounding article reference
* caption text
* crop coordinates
* extraction confidence
* model version

---

## 8. Politician Identification Methodology

### 8.1 Objective

Identify politicians referenced in articles and depicted in images, and optionally infer party or affiliation.

### 8.2 Reference data requirements

The system requires a maintained politician reference dataset containing:

* canonical names
* aliases
* parties
* offices
* tenure periods
* jurisdictions
* portrait references if available
* unique IDs
* source provenance

### 8.3 Name resolution

Text mentions must be normalized by:

1. Token normalization.
2. Alias matching.
3. Title stripping where appropriate.
4. Context-based disambiguation.
5. Historical tenure validation where applicable.

### 8.4 Identity resolution for text

A mention is linked to a politician entity only when one or more of these conditions holds:

1. Exact or near-exact name match plus role context.
2. Alias match plus political context.
3. Named role title consistent with reference data.
4. Co-mentioned office, party, or electorate aligns with a politician entity.
5. External reference dataset provides strong match evidence.

### 8.5 Identity resolution for images

An image is linked to a politician identity when:

1. Face embedding matches known reference portrait with sufficient confidence, or
2. Caption/context explicitly names the politician, or
3. Combined weak signals reach a policy threshold.

### 8.6 Affiliation inference

Affiliation may be inferred from:

* explicit party mentions
* office title and known office-holder records
* caption text
* article context
* reference dataset

Affiliation inference must never be asserted with higher confidence than the underlying evidence supports.

### 8.7 Ambiguity rules

If more than one politician is plausible:

* preserve all candidates
* mark as ambiguous
* do not collapse into a single identity without thresholded confidence
* expose candidate list in the data model

### 8.8 Temporal sensitivity

Politician identity and affiliation are time-sensitive. The reference dataset must support historical time ranges so that articles can be matched against the correct time period.

### 8.9 Bias and fairness controls

The system must:

* avoid assuming political significance based on image prominence alone
* avoid overfitting to party cues in certain newspapers
* evaluate false negatives across underrepresented groups and regions
* keep confidence calibrated by jurisdiction and source quality

---

## 9. Analytics & Metrics Definition

### 9.1 Required metric families

1. Document metrics.
2. Article metrics.
3. Image metrics.
4. Politician metrics.
5. Editorial composition metrics.
6. Quality metrics.
7. Confidence metrics.
8. Trend metrics.

### 9.2 Document metrics

* total pages
* total articles
* total photos
* total politician photos
* total OCR pages
* average page quality
* publication date
* processing completion time

### 9.3 Article metrics

* article count by topic
* article length
* article placement by page and section
* headline presence rate
* article confidence distribution
* politician mention count per article

### 9.4 Politician metrics

* number of mentions
* number of articles mentioning the politician
* number of image appearances
* newspaper frequency
* time trend
* co-mention graph degree
* affiliation distribution

### 9.5 Image metrics

* total images
* classified politician images
* uncertain images
* face detection rate
* identity match rate
* caption association rate

### 9.6 Quality metrics

* OCR confidence mean and distribution
* layout confidence mean and distribution
* article segmentation error estimate
* image classification uncertainty rate
* manual review rate
* duplicate detection rate

### 9.7 Confidence-weighted metrics

Any metric based on extraction should support:

* raw count
* confidence-weighted count
* strict-threshold count
* review-required count

### 9.8 Trend calculations

Trends must be computable by:

* date
* newspaper title
* edition
* jurisdiction
* politician
* party
* article type

### 9.9 Deduplication logic

Deduplication must support:

1. Same article across reprints.
2. Same image reused across pages.
3. Same politician mentioned multiple times in one article.
4. Same politician depicted in multiple crops from one source image.

---

## 10. API Specification (Endpoints, Inputs/Outputs)

### 10.1 API design principles

1. Versioned endpoints.
2. Pagination everywhere a list can grow.
3. Confidence and provenance included in all relevant responses.
4. No hidden server-side transformation without version markers.
5. Stable object IDs across requests.

### 10.2 Core endpoints

#### Documents

* `POST /v1/documents`

  * Input: PDF upload or source reference.
  * Output: document_id, run_id, status.

* `GET /v1/documents/{document_id}`

  * Output: document metadata and status.

* `GET /v1/documents/{document_id}/pages`

  * Output: page list.

#### Pages

* `GET /v1/pages/{page_id}`

  * Output: page metadata, render links, quality metrics.

* `GET /v1/pages/{page_id}/blocks`

  * Output: all page blocks and classifications.

#### Articles

* `GET /v1/articles`

  * Filters: date range, newspaper, topic, confidence, politician, page.
  * Output: article summaries.

* `GET /v1/articles/{article_id}`

  * Output: full article record, segments, confidence, provenance.

#### Images

* `GET /v1/images`

  * Filters: politician status, confidence, date, newspaper, face count.

* `GET /v1/images/{image_id}`

  * Output: image record, classification, matching evidence.

#### Entities

* `GET /v1/entities`

  * Filters: name, affiliation, is_politician, date, newspaper.

* `GET /v1/entities/{entity_id}`

  * Output: canonical entity record, mentions, affiliations.

#### Analytics

* `GET /v1/analytics/overview`

  * Output: summary metrics across selected filters.

* `GET /v1/analytics/politicians`

  * Output: politician trend metrics.

* `GET /v1/analytics/articles`

  * Output: article trend metrics.

* `GET /v1/analytics/images`

  * Output: image trend metrics.

#### Search

* `GET /v1/search`

  * Search text across extracted article bodies, captions, and entities.
  * Output: ranked results with source references.

#### Review

* `GET /v1/review/queue`

  * Output: items below confidence threshold.

* `POST /v1/review/{object_id}`

  * Input: reviewer annotation.
  * Output: updated review state.

### 10.3 API response requirements

Every response for extracted content must include:

* object ID
* source object IDs
* confidence
* model/version info
* timestamp
* provenance
* pagination where relevant

---

## 11. Dashboard Functional Requirements

### 11.1 Dashboard goals

The dashboard must allow users to:

* inspect processing status
* browse newspapers and issues
* review extracted articles
* inspect politician photos and mentions
* analyze trends and comparisons
* validate extraction quality

### 11.2 Required views

1. Overview dashboard.
2. Newspaper/issue explorer.
3. Article viewer.
4. Page viewer with overlay bounding boxes.
5. Politician profile page.
6. Image gallery with classification filters.
7. Quality and review dashboard.
8. Trend comparison dashboard.

### 11.3 Overview dashboard

Must show:

* total documents processed
* articles extracted
* politician mentions
* politician images
* OCR quality indicators
* exception rate
* pending review count

### 11.4 Drill-down behavior

Users must be able to:

* click a metric and trace to underlying documents
* open page images with overlays
* inspect article blocks
* inspect OCR text at word level
* view classification evidence

### 11.5 Visual controls

Must support:

* date filters
* newspaper filters
* politician filters
* confidence threshold filters
* article type filters
* page range filters

### 11.6 Review assistance

If manual review is enabled:

* highlight uncertain objects
* show top candidate identities
* allow confirm/reject/edit actions
* store reviewer identity and timestamp

### 11.7 Export

Users must be able to export:

* CSV summaries
* JSON extracts
* issue-level reports
* annotated review files

---

## 12. Storage Architecture (Databases, Indexing Strategy)

### 12.1 Storage tiers

1. Raw file store for original PDFs.
2. Derived artifact store for page images, OCR output, overlays, and embeddings.
3. Relational database for canonical records and analytics tables.
4. Search index for article and entity text search.
5. Vector index for face embeddings and semantic similarity if required.

### 12.2 Raw object storage

Store:

* original PDFs
* page render images
* cropped images
* OCR intermediate artifacts
* annotated overlays

Requirements:

* immutable
* versioned
* checksum-verified
* lifecycle-managed

### 12.3 Relational database

Use for:

* documents
* pages
* blocks
* articles
* images
* faces
* entities
* mentions
* runs
* metrics

### 12.4 Indexing strategy

Primary indexes:

* document_id
* page_id
* article_id
* image_id
* entity_id
* publication_date
* newspaper/title
* politician name
* confidence score
* processing run/version

Text indexes:

* article body
* captions
* normalized entity names

Vector indexes:

* face embeddings
* optional semantic embeddings for entity disambiguation

### 12.5 Partitioning strategy

Partition large tables by:

* publication date
* newspaper title
* batch/run ID
* document ID if needed

### 12.6 Retention

Must define:

* raw file retention
* derived artifact retention
* review annotation retention
* model output retention
* analytics retention

### 12.7 Lineage storage

Every derived object must reference:

* parent object IDs
* model version
* pipeline version
* transform parameters
* timestamp

---

## 13. Scalability & Performance Considerations

### 13.1 Scaling model

The architecture must support horizontal scaling of:

* ingestion workers
* OCR workers
* layout workers
* NLP workers
* vision workers
* analytics jobs
* API instances

### 13.2 Batch processing performance

Targets should be defined per deployment, but the system must support:

* document-level parallelism
* page-level parallelism
* independent model execution per stage
* retries at stage boundaries

### 13.3 Throughput control

Use queues between stages to:

* absorb spikes
* isolate slow OCR or vision stages
* prevent cascading failures
* allow backpressure

### 13.4 Large file handling

For long multi-page PDFs:

* process incrementally page by page
* avoid loading entire document into memory when unnecessary
* persist intermediate outputs early

### 13.5 Near-real-time extension

To extend into near-real-time:

* preserve same document/page/block schema
* route new documents into the same queue-based pipeline
* introduce priority queues if needed

### 13.6 Performance bottlenecks

Expected bottlenecks:

* OCR on scanned pages
* image embedding generation
* layout detection on dense pages
* search indexing at scale

---

## 14. Error Handling & Edge Cases

### 14.1 OCR errors

Handle:

* garbled words
* missing columns
* merged lines
* broken hyphenation
* incorrect reading order
* multilingual contamination

Required behavior:

* preserve confidence
* keep alternative text when available
* do not fabricate missing spans

### 14.2 Ambiguous layouts

Handle:

* wrap-around articles
* nested captions
* sidebars embedded in article flow
* ad/editorial boundary confusion
* unusual page grids

Required behavior:

* emit uncertain block labels
* preserve multiple segmentation hypotheses if needed
* choose one primary parse plus candidate alternatives when ambiguity is high

### 14.3 Duplicate detection

Handle duplicates at:

* article level
* image level
* entity mention level
* politician identity level

Rules:

* deduplicate only with explicit similarity thresholds
* preserve raw duplicates for audit
* mark whether a record is a canonical item or a derivative occurrence

### 14.4 False positives in image recognition

Handle:

* non-politician headshots
* generic event photos
* unrelated public figures
* group images
* logos mistaken as faces

Required behavior:

* return uncertainty rather than forced classification
* require corroborating evidence for final politician label where possible

### 14.5 Missing or low-quality images

Handle:

* clipped images
* blank regions
* compression artifacts
* low resolution scans
* partial page capture

Required behavior:

* keep region record
* lower confidence
* avoid identity assignment without evidence

### 14.6 Mixed-language articles

Handle:

* language switching
* transliterated names
* localized political titles

### 14.7 Partial failures

If one stage fails:

* mark affected downstream objects as incomplete
* retain all upstream artifacts
* allow stage re-run without re-ingestion

### 14.8 Corrupt PDFs

Handle:

* unreadable pages
* damaged streams
* malformed objects

Required behavior:

* preserve raw file
* record failure mode
* attempt page-level salvage where feasible

---

## 15. Security & Privacy Considerations

### 15.1 Security controls

1. Authenticate API access.
2. Authorize by role.
3. Audit all review edits.
4. Encrypt data at rest and in transit.
5. Restrict raw file and artifact access.

### 15.2 Data protection

Because newspaper content may include personal data:

* expose only required fields by role
* retain provenance for every extracted claim
* support deletion or suppression workflows if legally required

### 15.3 Model governance

Must track:

* model version
* training dataset version
* reference dataset version
* approval status
* known limitations

### 15.4 Abuse prevention

Prevent:

* unauthorized bulk export
* misuse of face identity data
* silent alteration of canonical outputs
* hidden model swaps without version record

### 15.5 Privacy boundary

The system processes published newspaper material, but still may handle:

* portraits
* names
* political affiliation
* contextual personal data

Therefore:

* keep access controlled
* log administrative actions
* enforce minimum necessary data exposure

---

## 16. Dependencies & External Systems

### 16.1 Required dependencies

1. PDF rendering engine.
2. OCR engine.
3. Layout detection model.
4. NER/entity linking model.
5. Face detection model.
6. Face embedding or image similarity component.
7. Politician reference dataset.
8. Search/indexing engine.
9. Database.
10. Object storage.
11. Job queue/orchestration system.

### 16.2 Optional external systems

1. External politician registry or biographical dataset.
2. Media archive metadata source.
3. Human review/annotation platform.
4. Authentication/SSO provider.
5. BI tool integration.

### 16.3 External dataset requirements

Politician reference data must include:

* canonical identifier
* name variants
* role history
* party history
* jurisdiction
* date ranges
* source provenance

### 16.4 Dependency constraints

The system must not require proprietary black-box components as a hard dependency unless explicitly chosen for deployment. If a proprietary model is used, a documented fallback path must exist.

---

## 17. Deployment Model (Local / Cloud Assumptions)

### 17.1 Supported deployment modes

1. Local workstation deployment.
2. On-premises server deployment.
3. Private cloud deployment.
4. Public cloud deployment.
5. Hybrid deployment.

### 17.2 Deployment architecture

The logical architecture remains identical across deployment modes:

* ingestion service
* processing workers
* storage services
* API service
* dashboard frontend
* queue/orchestration layer

### 17.3 Local deployment constraints

Local mode should:

* support limited batch volumes
* run a reduced worker count
* avoid dependence on managed cloud-only services
* allow offline processing if models are locally available

### 17.4 Cloud deployment constraints

Cloud mode should:

* scale workers dynamically
* use managed object storage and databases if desired
* support secure artifact access
* isolate processing and serving layers

### 17.5 Reproducibility across environments

To ensure reproducibility:

* pin model versions
* pin preprocessing parameters
* pin pipeline version
* store reference dataset versions
* store run configuration with every output

---

# Validation

## Internal consistency check

1. Every output artifact is traceable to a PDF document and page.
2. Every page artifact traces to a raw source file and processing run.
3. Every article, image, mention, and metric is linked to upstream artifacts.
4. Deterministic and probabilistic steps are separated.
5. Versioning exists at document, pipeline, model, and dataset levels.
6. Confidence is attached to all non-deterministic outputs.
7. Edge-case handling is defined for OCR, layout, identity resolution, and duplicates.

## Missing subsystem check

No subsystem omitted:

1. Ingestion Layer
2. Preprocessing
3. Content Segmentation
4. Entity Extraction
5. Image Analysis
6. Data Structuring & Storage
7. Analytics Engine
8. API Layer
9. Frontend Dashboard

## Ambiguities and explicit assumptions

### Ambiguities

1. Exact politician reference source is unspecified.
2. Exact OCR engine is unspecified.
3. Exact cloud provider or local stack is unspecified.
4. Exact manual review workflow is unspecified.
5. Ground truth availability for training is unspecified.

### Working assumptions

1. A maintained politician reference dataset will be provided or constructed.
2. OCR and CV models will be deployable in the target environment.
3. Human review is optional but supported.
4. Training data for layout and classification can be produced from samples.
5. Historical affiliation matching is required, not just current affiliation.

## Technical infeasibility constraints and closest alternatives

1. Perfect politician identification from images is infeasible without a reference image dataset and sufficient image quality. Closest viable alternative: confidence-scored matching with ambiguity states and human review.
2. Perfect article segmentation in all newspaper layouts is infeasible. Closest viable alternative: hybrid layout-model + deterministic rule system with confidence and review fallback.
3. Perfect OCR on all scans is infeasible. Closest viable alternative: confidence-scored OCR plus page-level quality estimation and selective reprocessing.

## Confidence level per major component

1. Ingestion Layer: High
2. Preprocessing/OCR Layer: High for architecture, medium for output quality
3. Layout Detection: Medium
4. Content Segmentation: Medium
5. Entity Extraction: High
6. Image Analysis: Medium
7. Politician Identification: Medium to low without reference imagery, medium with reference imagery
8. Storage Architecture: High
9. Analytics Layer: High
10. API Layer: High
11. Dashboard Requirements: High
12. Reproducibility and lineage: High

## Final completeness statement

All required subsystems are specified, data flows are defined end to end, deterministic and probabilistic logic are separated, storage and lineage requirements are explicit, error cases are covered, and the architecture is implementation-ready for batch processing with a path to near-real-time operation.
