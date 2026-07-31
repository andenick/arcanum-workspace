---
name: hdarp-integrate
description: "Safely integrate, catalog, and organize HDARP-extracted content into a project's Knowledge Base."
when-to-use: '"User needs to organize HDARP extractions, catalog documents, or sync results to a Knowledge Base"'
search-hints: "hdarp integrate catalog organize knowledge base sync audit"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit
argument-hint: "[action] [target]"
requires: hdarp-extract
part-of: HDARP Framework v6.3 (cloud Claude Read-tool — distinct from /hopper local VLM engine)
---

# HDARP Integration Skill

> **Scope note**: Hopper-produced KB outputs (`hdarp`) have the SAME 4-artifact shape as HDARP outputs (FULL_TEXT_*.md, CSV_Tables/, equations/, figures/) and can be integrated via the same Phase 7 pipeline. The DISTINCTION is upstream (Hopper = local VLM, HDARP = cloud Claude Read-tool); the integration step is engine-agnostic. Both engines also write to `_UNIFIED/UNIFIED_RENAME_HISTORY.csv` when their outputs trigger renames.


## Description

Safely integrate, catalog, and organize HDARP-extracted content from any project, with unification into Robert's centralized Knowledge_Base. This skill provides a generalizable workflow for organizing HDARP extractions across all Arcanum projects.

## Usage

```
/hdarp-integrate [project] [phase] [options]
```

**Parameters**:
- `project`: Project name (Volcker, USSR, Kalendern, etc.) or ALL
- `phase`: backup | audit | catalog | classify | crossref | integrate | robert-sync | status | verify
- `options`: batch ranges, --dry-run, --force, etc.

**Examples**:
```bash
/hdarp-integrate status                    # Show all project integration status
/hdarp-integrate Volcker backup            # Phase 1: Full Knowledge_Base archive
/hdarp-integrate Volcker audit             # Phase 2: Audit all documents
/hdarp-integrate Volcker audit 1-100       # Phase 2: Audit documents 1-100
/hdarp-integrate Volcker catalog           # Phase 3: Create information catalogs
/hdarp-integrate Volcker classify          # Phase 4: Apply classifications
/hdarp-integrate Volcker crossref          # Phase 5: Map cross-references
/hdarp-integrate Volcker integrate         # Phase 6: Create project integration
/hdarp-integrate Volcker robert-sync       # Phase 7: Sync to Robert unified KB
/hdarp-integrate ALL robert-sync           # Sync all projects to Robert
/hdarp-integrate robert-status             # Show Robert unified KB status
/hdarp-integrate Volcker verify backup     # Verify archive integrity
/hdarp-integrate Volcker verify catalog    # Verify catalog completeness
```

## HDARP Lifecycle Position

This skill is step **5** (INTEGRATE) in the HDARP lifecycle:
1. `/hdarp-campaign` — campaign setup
2. `/preparehdarp` — chunk and prepare PDFs
3. `/sphdarp` (or variants) — extract all 4 content types via agents
4. `/hdarp-wrapup` — validate, remediate, document, close out
5. **`/hdarp-integrate`** — catalog, classify, crossref, Robert sync

**Prerequisites**: All batches in scope must be VERIFIED (via `/hdarp-wrapup`) before running integration.

## Architecture

```
the workspace
|-- Projects
|   |-- Volcker/
|   |   |-- Knowledge_Base              # Project HDARP outputs
|   |   |-- HDARP_Integration # Integration catalogs
|   |   `-- Archives/                    # Timestamped backups
|   |-- USSR/
|   `-- {Project}/
|
`-- Robert
    `-- Knowledge_Base                  # UNIFIED REPOSITORY
        |-- _UNIFIED/                    # Cross-project catalogs
        |   |-- UNIFIED_CATALOG.csv
        |   |-- UNIFIED_ENTITY_INDEX.csv
        |   |-- UNIFIED_TABLE_INDEX.csv
        |   `-- PROJECT_REGISTRY.json
        |
        |-- Volcker/                     # Namespace: Volcker project
        |   |-- HDARP_Integration/       # Synced integration catalogs
        |   `-- Documents/               # Synced document directories
        |
        `-- {Project}/                   # Any additional project
```

## Design Principles

1. **Project-Agnostic**: Works with any HDARP campaign
2. **Robert-Centric**: All reads ultimately sync to Robert's Knowledge_Base
3. **Resumable**: State tracking allows interruption and continuation
4. **Non-Destructive**: Backup before any modifications
5. **v2.0 PDF Library**: Source PDFs land in `_PDF_LIBRARY/<Project>/` (flat) with content-type symlinks in `_BY_CONTENT_TYPE/`. Use `/robert-pdf-sync <project>` or call `_UNIFIED/_integration_audit/migrate_project_to_robert.py` directly. Standard: `ROBERT_PDF_LIBRARY_V2_STANDARD.md`.

---

## Phase 1: BACKUP (Zip-Based KB Archival)

**Goal**: Create a zip archive of the finalized Knowledge_Base after campaign completion, before integration modifies anything.

**Path**: `KB_CAMPAIGN_{YYYYMMDD}.zip`

Each campaign produces a separate zip file. Existing archives are never modified.

### Execution Steps

1. **Detect Project Path**:
   ```
   PROJECT_PATH = {project}
   KB_PATH = {PROJECT_PATH}/Knowledge_Base
   ARCHIVES_DIR = {PROJECT_PATH}/Archives/
   ZIP_FILE = {ARCHIVES_DIR}/KB_CAMPAIGN_{YYYYMMDD}.zip
   ```

2. **Create Archives Directory** (if needed):
   ```bash
   mkdir -p "{ARCHIVES_DIR}"
   ```

3. **Create Zip Archive of Knowledge_Base**:
   ```python
   import zipfile, os
   with zipfile.ZipFile(ZIP_FILE, 'w', zipfile.ZIP_DEFLATED) as zf:
       for root, dirs, files in os.walk(KB_PATH):
           for f in files:
               full = os.path.join(root, f)
               arcname = os.path.relpath(full, PROJECT_PATH)
               zf.write(full, arcname)
   ```

4. **Include Critical State Files in Archive**:
   Add to the same zip:
   - `BATCH_STATE.json` (from project root or HDARP_Processing)
   - `HDARP_MASTER_CATALOG.csv` (from KB root)
   - `HDARP_Processing` directory contents

5. **Generate BACKUP_MANIFEST.json** (written inside the Archives/ directory alongside the zip):
   ```json
   {
     "project": "Volcker",
     "backup_date": "2026-02-17T12:00:00Z",
     "backup_type": "zip",
     "zip_file": "KB_CAMPAIGN_20260217.zip",
     "source_path": "Knowledge_Base",
     "statistics": {
       "total_files": 140220,
       "total_size_bytes": 9663676416,
       "zip_size_bytes": 4200000000
     }
   }
   ```

6. **Verify Zip Integrity**:
   ```python
   with zipfile.ZipFile(ZIP_FILE, 'r') as zf:
       bad = zf.testzip()
       assert bad is None, f"Corrupt file in archive: {bad}"
       file_count = len(zf.namelist())
   ```

### Output

- `{ARCHIVES_DIR}/KB_CAMPAIGN_{YYYYMMDD}.zip` - Complete KB zip archive
- `{ARCHIVES_DIR}/BACKUP_MANIFEST.json` - Archive inventory and metadata
- Subsequent campaigns create new zip files (never modify existing archives)

---

## Phase 2: AUDIT (Document-Level Catalog)

**Goal**: Verify and catalog every document directory

### Per-Document Checks

For each directory in Knowledge_Base:
- [ ] Directory exists with valid name
- [ ] FULL_TEXT.md present (primary, Sraffa 4.0 assembly) or Text/ directory (legacy)
- [ ] CSV_Tables/ directory (if applicable)
- [ ] DOCUMENT_COMPLETE.md or equivalent marker
- [ ] Quality score documented
- [ ] Batch assignment tracked

### Execution Steps

1. **Enumerate All Document Directories**:
   ```python
   documents = list(KB_PATH.glob("*/"))  # Top-level document dirs
   ```

2. **Audit Each Document**:
   ```csv
   document_id,document_name,path,has_full_text,has_text_dir,has_csv_tables,has_completion_marker,quality_score,batch_id,status
   001,1953_FDIC_Annual_Report,1953_FDIC_Annual_Report,TRUE,TRUE,TRUE,TRUE,25.2,B001,COMPLETE
   002,1963_Friedman_Schwartz,1963_Friedman_Schwartz,TRUE,TRUE,FALSE,TRUE,24.8,B002,COMPLETE
   ```

3. **Identify Gaps**:
   - Documents missing FULL_TEXT.md
   - Documents missing completion markers
   - Documents with low quality scores

### Output Files

```
{PROJECT}/HDARP_Integration
|-- DOCUMENT_AUDIT.csv         # Full document inventory
|-- COMPLETION_GAPS.md         # Documents needing attention
`-- QUALITY_REPORT.md          # Quality distribution analysis
```

### DOCUMENT_AUDIT.csv Schema

```csv
document_id,document_name,source_path,created_date,modified_date,has_full_text,has_text_dir,text_file_count,has_csv_tables,csv_count,has_images,image_count,has_equations,equation_count,has_figures,figure_count,has_completion_marker,quality_score,quality_score_max,batch_id,batch_round,processing_date,source_pdf,page_count,word_count,status,notes
```

---

## Phase 3: INFORMATION CATALOG (Content-Level)

**Goal**: Catalog every piece of extracted information

### 3a. TABLE_CATALOG.csv

Catalog all extracted tables across all documents.

```csv
table_id,document_id,document_name,chunk_num,table_num,title,description,rows,columns,page,file_path,extraction_quality,has_headers,data_types
TBL_00001,001,1953_FDIC_Annual_Report,01,01,District Offices,List of FDIC district offices,12,4,6,CSV_Tables/chunk_01_table_01_district_offices.csv,HIGH,TRUE,string|string|int|string
TBL_00002,001,1953_FDIC_Annual_Report,01,02,Annual Statistics,Yearly banking statistics,25,8,8,CSV_Tables/chunk_01_table_02_annual_statistics.csv,HIGH,TRUE,int|float|float|float|int|int|int|string
```

**Extraction Method**:
1. Scan all `CSV_Tables/` directories
2. Parse each CSV for metadata (rows, columns, headers)
3. Extract title from filename or first row
4. Assign unique table_id

### 3b. EQUATION_CATALOG.csv

Catalog all extracted equations.

```csv
equation_id,document_id,document_name,chunk_num,equation_num,latex,description,variables,page,file_path,complexity_level
EQ_00001,045,1974_Merton_Pricing_Corporate_Debt,03,A.8,dV = (\alpha V - C)dt + \sigma V dz,Firm value dynamics,V=firm value; alpha=drift; C=coupon; sigma=volatility,45,Equations/chunk_03_equations.md,HIGH
```

**Extraction Method**:
1. Scan all `Equations/` directories and `equations.md` files
2. Parse LaTeX blocks
3. Extract variable definitions from surrounding context

### 3c. FIGURE_CATALOG.csv

Catalog all extracted figures.

```csv
figure_id,document_id,document_name,chunk_num,figure_num,title,chart_type,x_axis,y_axis,page,description_file,image_file,description_words
FIG_00001,022,1963_Friedman_Schwartz,02,3,Money Stock 1867-79,time_series,Year,Money Stock (USD millions),30,Figures/chunk_02_figures.md,Images/figure_3.png,245
```

**Chart Types**: time_series, bar_chart, scatter_plot, pie_chart, histogram, flowchart, diagram, table_figure, map, other

### 3d. ENTITY_CATALOG.csv - COMPREHENSIVE EXTRACTION

Extract all named entities from documents.

**Entity Types**:
- **PERSON**: All named individuals (Paul Volcker, Janet Yellen, etc.)
- **ORGANIZATION**: All institutions, agencies, banks, firms
- **REGULATION**: All regulations, rules, acts, standards (Basel III, Dodd-Frank, etc.)
- **DATE**: All significant dates, time periods, deadlines
- **LOCATION**: Countries, cities, regions mentioned
- **FINANCIAL_INSTRUMENT**: Specific instruments, products, metrics
- **ACRONYM**: All acronyms with expansions (RWA, LCR, NSFR, etc.)

```csv
entity_id,entity_type,entity_name,normalized_name,document_ids,document_count,first_mention_doc,first_mention_page,total_frequency,context_snippet,related_entities,wikidata_id
ENT_00001,ORGANIZATION,Federal Reserve System,Federal Reserve,001|002|003|045,4,001,1,847,"The Federal Reserve announced...",ENT_00002|ENT_00045,Q53536
ENT_00002,PERSON,Paul Volcker,Paul A. Volcker,004|005|022,3,004,3,156,"Chairman Volcker testified...",ENT_00001,Q319843
ENT_00003,REGULATION,Basel III,Basel III Capital Framework,006|007|008,3,006,1,423,"Basel III requirements mandate...",ENT_00001|ENT_00050,Q582578
ENT_00004,DATE,2008-09-15,2008-09-15,009|010,2,009,2,89,"Lehman Brothers filed bankruptcy on...",ENT_00100,
ENT_00005,ACRONYM,RWA,Risk-Weighted Assets,001|002|022,3,001,1,1247,"Risk-Weighted Assets (RWA) are calculated...",ENT_00003,
```

**Extraction Method**:
1. Parse FULL_TEXT.md (primary source — Sraffa 4.0 assembled) and Text/*.md files (legacy fallback)
2. Apply NLP pattern matching:
   - Proper noun capitalization patterns
   - Organization suffixes (Inc., Corp., Bank, Agency)
   - Regulation patterns (Act, Rule, Regulation, Standard)
   - Date patterns (YYYY-MM-DD, Month DD, YYYY)
   - Acronym patterns (ALL CAPS followed by expansion)
3. Normalize entities (merge variants)
4. Calculate cross-document frequency

---

## Phase 4: CLASSIFICATION (Taxonomy)

**Goal**: Apply consistent classification to all documents

### Classification Dimensions

#### 1. Source Type (15 categories)
```
Regulatory_Fed, Regulatory_NYFed, Regulatory_FDIC, Regulatory_SEC,
International_BIS, International_IMF, International_Other,
Academic_NBER, Academic_Other, Industry_BPI, Industry_Consulting,
Bank_Filing, MUFG, Regulatory_EU, Regulatory_UK, Other
```

#### 2. Topic (10 categories)
```
Capital_Regulation, Stress_Testing, Credit_Risk,
Financial_Stability, Climate_Risk, Liquidity,
Bank_Supervision, Central_Bank_Reports, Academic_Research, Other
```

#### 3. Temporal Period
```
Pre_Basel (pre-1988), Basel_I_Era (1988-2003), Basel_II_Era (2004-2010),
Post_Crisis (2011-2018), Current (2019-present)
```

#### 4. Content Type
```
Regulatory_Rule, Working_Paper, Annual_Report, Earnings_Report,
Academic_Paper, Policy_Brief, Technical_Note, Data_Supplement,
Historical_Document, Statistical_Release, Guidance, Speech, Other
```

### Output: CLASSIFICATION_MASTER.csv

```csv
document_id,document_name,source_type,source_organization,topic_primary,topic_secondary,temporal_period,temporal_year,content_type,geographic_scope,language,classification_confidence,classified_by,classification_date
001,1953_FDIC_Annual_Report,Regulatory_FDIC,FDIC,Bank_Supervision,Financial_Stability,Pre_Basel,1953,Annual_Report,US,en,0.95,hdarp-integrate,2026-02-17
002,1974_Merton_Pricing_Corporate_Debt,Academic_Other,Journal of Finance,Credit_Risk,Capital_Regulation,Pre_Basel,1974,Academic_Paper,Global,en,0.92,hdarp-integrate,2026-02-17
```

---

## Phase 5: CROSS-REFERENCE INDEX

**Goal**: Map relationships between documents

### Relationship Types

- **supersedes / superseded_by**: Version relationships (2024 rule supersedes 2023 rule)
- **cites / cited_by**: Reference relationships (paper A cites paper B)
- **implements / implemented_by**: Regulation to implementation
- **related_to**: Topical similarity (same subject matter)
- **same_source**: Same author/organization
- **same_entity**: Mentions same key entities
- **responds_to / response_from**: Comment letters, responses

### Output: CROSS_REFERENCE_INDEX.json

```json
{
  "version": "1.0",
  "generated": "2026-02-17T12:00:00Z",
  "project": "Volcker",
  "document_count": 2568,
  "relationship_count": 8432,
  "relationships": [
    {
      "source_id": "001",
      "target_id": "002",
      "relationship": "cites",
      "confidence": 0.95,
      "evidence": "Page 45: 'As noted by Friedman and Schwartz (1963)...'"
    },
    {
      "source_id": "045",
      "target_id": "044",
      "relationship": "supersedes",
      "confidence": 1.0,
      "evidence": "Document title indicates revision"
    }
  ],
  "entity_links": [
    {
      "entity_id": "ENT_00001",
      "entity_name": "Federal Reserve",
      "document_ids": ["001", "002", "003", "045"],
      "document_count": 4
    }
  ]
}
```

---

## Phase 6: PROJECT INTEGRATION

**Goal**: Create unified search-ready catalog for the project

### Integration Directory Structure

```
{Project}/HDARP_Integration
|-- INTEGRATION_MANIFEST.json     # Project stats and metadata
|-- INTEGRATION_STATE.json        # Processing state (resumable)
|-- DOCUMENT_CATALOG.csv          # All documents (from Phase 2)
|-- TABLE_CATALOG.csv             # All tables (from Phase 3a)
|-- EQUATION_CATALOG.csv          # All equations (from Phase 3b)
|-- FIGURE_CATALOG.csv            # All figures (from Phase 3c)
|-- ENTITY_CATALOG.csv            # Extracted entities (from Phase 3d)
|-- CLASSIFICATION_MASTER.csv     # All classifications (from Phase 4)
|-- CROSS_REFERENCE_INDEX.json    # Document relationships (from Phase 5)
|-- QUALITY_SUMMARY.md            # Quality metrics overview
`-- INTEGRATION_COMPLETE.md       # Final report
```

### INTEGRATION_MANIFEST.json

```json
{
  "project": "Volcker",
  "version": "1.0",
  "integration_date": "2026-02-17T12:00:00Z",
  "phases_completed": ["backup", "audit", "catalog", "classify", "crossref", "integrate"],
  "statistics": {
    "documents": {
      "total": 2568,
      "complete": 2450,
      "partial": 100,
      "needs_attention": 18
    },
    "tables": {
      "total": 47153,
      "high_quality": 45000,
      "needs_review": 2153
    },
    "equations": {
      "total": 3421,
      "with_variables": 2890
    },
    "figures": {
      "total": 22569,
      "with_descriptions": 22000
    },
    "entities": {
      "total": 15234,
      "persons": 2341,
      "organizations": 5678,
      "regulations": 1234,
      "dates": 3456,
      "acronyms": 2525
    }
  },
  "quality_metrics": {
    "average_quality_score": 25.2,
    "quality_score_max": 27,
    "documents_above_threshold": 2450,
    "threshold": 22
  },
  "robert_sync_status": "pending"
}
```

---

## Phase 7: ROBERT SYNC (Unified Repository)

**Goal**: Sync all integrated HDARP content to Robert's unified Knowledge_Base

### Robert Structure

```
Knowledge_Base
|-- _UNIFIED/                          # Cross-project catalogs
|   |-- UNIFIED_CATALOG.csv            # All projects, all documents
|   |-- UNIFIED_ENTITY_INDEX.csv       # Entities across all projects
|   |-- UNIFIED_TABLE_INDEX.csv        # Tables across all projects
|   |-- UNIFIED_FIGURE_INDEX.csv       # Figures across all projects
|   |-- UNIFIED_EQUATION_INDEX.csv     # Equations across all projects
|   |-- PROJECT_REGISTRY.json          # Registered projects
|   `-- UNIFIED_STATE.json             # Unified sync state
|
|-- Volcker/                           # Namespace: Volcker project
|   |-- HDARP_Integration/             # Synced integration catalogs
|   |   |-- DOCUMENT_CATALOG.csv
|   |   |-- TABLE_CATALOG.csv
|   |   |-- ENTITY_CATALOG.csv
|   |   |-- CLASSIFICATION_MASTER.csv
|   |   |-- CROSS_REFERENCE_INDEX.json
|   |   `-- INTEGRATION_MANIFEST.json
|   |-- Documents/                     # Synced document directories
|   |   |-- 1953_FDIC_Annual_Report/
|   |   |-- 1963_Friedman_Schwartz/
|   |   `-- .../
|   `-- SYNC_REPORT_YYYYMMDD.md        # Sync completion report
|
`-- {Project}/                         # Additional projects follow same pattern
```

### Sync Actions

1. **Register Project**:
   ```json
   // PROJECT_REGISTRY.json
   {
     "projects": [
       {
         "name": "Volcker",
         "source_path": "Volcker",
         "registered_date": "2026-02-17T12:00:00Z",
         "last_sync": "2026-02-17T14:30:00Z",
         "document_count": 2568,
         "status": "synced"
       }
     ]
   }
   ```

2. **Copy Integration Catalogs**:
   ```bash
   cp -r "{Project}/HDARP_Integration" "Robert/HDARP_Integration"
   ```

3. **Sync Document Directories** (with deduplication):
   - Copy document folders to Robert
   - Apply compound key: `{project}_{document_id}`
   - Handle duplicates:
     - Identical checksum: Skip (already synced)
     - Different checksum: Create version suffix `_v2`, `_v3`

4. **Update Unified Catalog**:
   - Merge project catalog into `_UNIFIED/UNIFIED_CATALOG.csv`
   - Add `project` column prefix to all IDs

5. **Update Entity Index**:
   - Merge entities into `_UNIFIED/UNIFIED_ENTITY_INDEX.csv`
   - Track cross-project entity occurrences

6. **Generate Sync Report**:
   ```markdown
   # Volcker Sync Report - 2026-02-17

   ## Summary
   - Documents synced: 2,568
   - New documents: 2,568
   - Updated documents: 0
   - Skipped (identical): 0

   ## Unified Repository Status
   - Total projects: 1
   - Total documents: 2,568
   - Total entities: 15,234
   - Total tables: 47,153
   ```

### Deduplication Rules

```python
# Document identified by compound key
compound_key = f"{project}_{document_id}"

# If document exists in Robert:
if exists_in_robert(compound_key):
    source_checksum = compute_md5(source_path)
    robert_checksum = compute_md5(robert_path)

    if source_checksum == robert_checksum:
        action = "SKIP"  # Already synced
    else:
        # Create versioned copy
        version = get_next_version(compound_key)
        new_key = f"{compound_key}_v{version}"
        action = "CREATE_VERSION"
else:
    action = "COPY"
```

### Sync Commands

```bash
/hdarp-integrate Volcker robert-sync              # Sync single project
/hdarp-integrate Volcker robert-sync --dry-run    # Preview what would sync
/hdarp-integrate ALL robert-sync                  # Sync all projects
/hdarp-integrate robert-status                    # Show Robert unified KB status
/hdarp-integrate robert-verify                    # Verify Robert unified KB integrity
```

---

## State Tracking

### Per-Project State: INTEGRATION_STATE.json

```json
{
  "project": "Volcker",
  "version": "1.0",
  "phases_complete": ["backup", "audit"],
  "current_phase": "catalog",
  "current_batch": 5,
  "total_batches": 26,
  "documents_processed": 450,
  "documents_total": 2568,
  "robert_synced": false,
  "last_updated": "2026-02-17T12:00:00Z",
  "errors": [],
  "warnings": [
    {"document": "doc_123", "message": "Missing FULL_TEXT.md", "phase": "audit"}
  ]
}
```

### Robert Unified State: UNIFIED_STATE.json

```json
{
  "version": "1.0",
  "projects_registered": ["Volcker", "USSR"],
  "total_documents": 3200,
  "total_entities": 18500,
  "total_tables": 55000,
  "last_sync": {
    "Volcker": "2026-02-17T12:00:00Z",
    "USSR": "2026-02-15T09:00:00Z"
  },
  "last_updated": "2026-02-17T12:00:00Z"
}
```

---

## Batch Processing Strategy

### Configuration

- **Batch Size**: 100 documents per batch
- **Checkpointing**: State saved after each batch
- **Resumable**: Can resume from any batch if interrupted

### Execution

```bash
# Process specific batch range
/hdarp-integrate Volcker catalog 1-100     # Documents 1-100
/hdarp-integrate Volcker catalog 101-200   # Documents 101-200

# Auto-continue from last checkpoint
/hdarp-integrate Volcker catalog --continue
```

---

## Verification Commands

```bash
# Per-Phase Verification
/hdarp-integrate Volcker verify backup     # Compare archive to source
/hdarp-integrate Volcker verify audit      # Validate audit completeness
/hdarp-integrate Volcker verify catalog    # Check catalog integrity

# Robert Verification
/hdarp-integrate robert-verify             # Full unified KB integrity check
/hdarp-integrate robert-verify --checksums # Include checksum verification
```

### Verification Checks

1. **Backup Verification**:
   - Compare source and archive file counts
   - Verify random file checksums (sample 1%)
   - Confirm BACKUP_MANIFEST.json completeness

2. **Audit Verification**:
   - All documents in DOCUMENT_AUDIT.csv
   - No duplicate document_ids
   - Status fields populated

3. **Catalog Verification**:
   - All tables in TABLE_CATALOG.csv have valid paths
   - Entity IDs are unique
   - Cross-references resolve

4. **Robert Verification**:
   - All registered projects have synced directories
   - UNIFIED_CATALOG.csv matches project catalogs
   - No orphaned documents

---

## Error Handling

### Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| `PROJECT_NOT_FOUND` | Invalid project name | Check `Projects` |
| `KB_NOT_FOUND`| Missing Knowledge_Base | Run`/preparehdarp` first |
| `BACKUP_FAILED` | Disk space or permissions | Check disk space, permissions |
| `STATE_CORRUPTED` | Invalid state JSON | Restore from backup or reset |
| `SYNC_CONFLICT` | Document version mismatch | Use `--force` or resolve manually |

### Error Recovery

```bash
# Reset integration state (keeps backups)
/hdarp-integrate Volcker reset

# Force overwrite (use with caution)
/hdarp-integrate Volcker robert-sync --force

# Restore from backup
/hdarp-integrate Volcker restore 20260217
```

---

## Integration with Other Skills

- **After**: `/phdarp`, `/sphdarp`, `/pdarp`, `/spdarp` (when HDARP processing complete)
- **Before**: `/handoff` (document integration status)
- **Uses**: `/preparehdarp` artifacts (manifest.json, chunk structure)
- **Updates**: Robert's Knowledge_Base

---

## Success Criteria

### Per-Project Integration

- [ ] Full archive created with verified integrity
- [ ] All documents cataloged (DOCUMENT_CATALOG.csv)
- [ ] All tables cataloged (TABLE_CATALOG.csv)
- [ ] All equations cataloged (EQUATION_CATALOG.csv)
- [ ] All figures cataloged (FIGURE_CATALOG.csv)
- [ ] Comprehensive entity extraction (ENTITY_CATALOG.csv)
- [ ] Classification applied (CLASSIFICATION_MASTER.csv)
- [ ] Cross-references mapped (CROSS_REFERENCE_INDEX.json)
- [ ] INTEGRATION_COMPLETE.md generated

### Robert Unified Repository

- [ ] Robert Knowledge_Base structure created
- [ ] PROJECT_REGISTRY.json tracks all projects
- [ ] UNIFIED_CATALOG.csv contains all documents
- [ ] UNIFIED_ENTITY_INDEX.csv enables cross-project search
- [ ] Each project synced with proper namespacing
- [ ] Deduplication working correctly
- [ ] Cross-project search functional

---

## Quick Reference

| Command | Phase | Description |
|---------|-------|-------------|
| `/hdarp-integrate status` | - | Show all project status |
| `/hdarp-integrate {P} backup` | 1 | Create full Knowledge_Base archive |
| `/hdarp-integrate {P} audit` | 2 | Audit all documents |
| `/hdarp-integrate {P} catalog` | 3 | Create information catalogs |
| `/hdarp-integrate {P} classify` | 4 | Apply classifications |
| `/hdarp-integrate {P} crossref` | 5 | Map cross-references |
| `/hdarp-integrate {P} integrate` | 6 | Create project integration |
| `/hdarp-integrate {P} robert-sync` | 7 | Sync to Robert |
| `/hdarp-integrate ALL robert-sync` | 7 | Sync all projects |
| `/hdarp-integrate robert-status` | - | Show unified KB status |

---

**Skill Version**: 1.1
**Status**: PRODUCTION READY
**Created**: 2026-02-17
**Updated**: 2026-04-30
**Dependencies**: HDARP v4.5, Sraffa 4.0, Robert Knowledge_Base
**Note**: Sraffa 4.0 produces FULL_TEXT.md as primary output (replaces Text/ directory). Entity extraction should parse FULL_TEXT.md first, falling back to Text/*.md for legacy documents.

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
