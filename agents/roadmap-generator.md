---
name: roadmap-generator
description: Analyze learning materials and generate a structured learning roadmap. Must read all materials completely before generating roadmap.
---

# Roadmap Generator Agent

Analyzes learning materials and generates a structured learning roadmap.

## ⚠️ Language Requirement

**Must generate all H2/H3 titles and knowledge point names in the output language (English or 中文) specified by the main workflow**, as well as all dialogue with the user. Roadmap filename examples: `Learning Roadmap v1 - {Topic}.md` (English) / `学习路线图 v1 - {主题}.md` (中文).

## ⚠️ Core Requirement: Must Read All Learning Materials Completely

**Absolutely forbidden to generate a learning roadmap based on partial content.**

### Reading Strategy (Priority Order)

1. **Prefer single-pass full reads**
   - Directly read the entire file content
   - Report total file size / total page count
   - Only consider batching when encountering technical limits

2. **Auto-batch (no user confirmation needed)**
   - If a file is too large (e.g., PDF > 100 pages), auto-batch at 50 pages per batch
   - **Auto-continue; do not require user confirmation per batch**
   - Report progress after each batch

3. **Progress report format**
   ```
   📖 Reading: filename.pdf

   Progress: [████████░░░░░░░░░░░░] 150/329 pages (46%)

   ✅ Reading complete! 329/329 pages (100%)
   ```

## Input

- User-confirmed vault path
- User-selected learning file list (filenames and paths)
- `incremental_mode` (boolean, set by Phase 0)
- Previous roadmap file path (if incremental mode, the existing `Learning Roadmap v1 - {Topic}.md`)

## Reading Flow

### Markdown, Word, PPT, HTML, TXT Files
- Directly `Read` tool read of full content
- Word and PowerPoint files may be read as plain text in some environments; the Agent extracts the best available text
- Report file line count and size

### PDF, Word, PPT, HTML, TXT Files
- **Prefer single-pass read** (specify full page range)
- If it fails (exceeds limit), auto-batch:
  - 50 pages per batch
  - Auto-continue reading the next batch
  - Report progress per batch
- Aggregate all content on completion

### Progress Report Format

```
📖 Reading: Digital Transformation Roadmap.pdf

Progress: [████████████████████] 329/329 pages (100%)
✅ Reading complete!

---

📖 Reading: Strategic Management.md
✅ Reading complete! 156 pages

---

📖 All learning materials read:
- Digital Transformation Roadmap.pdf - 329 pages
- Strategic Management.md - 156 pages
Total: 2 files, 485 pages
```

## Output Files

After Phase 1 completes, **two** learning roadmap files must be saved:

### 1. Full Version (Must Save)

File: `Learning Roadmap (Full) v1 - {Topic}.md` (English) / `学习路线图（完整版） v1 - {主题}.md` (中文)

Contains:
- **Detailed description** of all knowledge points (200+ words)
- Complete case descriptions
- Accurate original citations
- Source page numbers
- **`source_range` annotation** (each knowledge point tagged with source file + page ranges, used by Phase 3's `context-extractor.py`)

`source_range` format specification:
```
source_range: {filename}:{page_range}, {page_range}, ...
```

**Page range supports three forms**:
- Single page: `102`
- Continuous range: `12-15`
- Mixed: `12-15, 45-48, 102`

**Multi-file coexistence** (one knowledge point spans multiple source files):
```
source_range: Digital Transformation.pdf:12-15, Strategy.md
```
`.md` / `.txt` files without page numbers indicate **full text** (non-PDF files have no page concept).

**Complex example**:
```
**Platform Governance Mechanisms**
  source_range: Platform Strategy.pdf:156-162, 203-205, Appendix.md
```
→ Indicates this knowledge point covers PDF pp.156-162 and pp.203-205, plus the full Appendix.md.

**⚠️ source_range is mandatory**: every knowledge point must have one. This annotation is used by `scripts/context-extractor.py` for automated extraction.

Format:
```markdown
# Learning Roadmap (Full) v1: {Topic Name}

This is a learning roadmap on {Topic Name}, compiled from {N} learning resources.

**Source Materials:**
- {Filename 1} - {pages/word count}
- {Filename 2} - {pages/word count}

---

## 01. First Category

### Topic One

**Knowledge Point 1**  `source_range: {filename}:{start_page}-{end_page}`
- Detailed explanation: this is the core concept of..., to be understood by beginners...
- Case: for example, the XX case from the learning material illustrates...
- Original citation: the learning material states "..." [[Source: Document Name, p.X]]

**Knowledge Point 2**  `source_range: {filename}:{start_page}-{end_page}, {start_page}-{end_page}`
- Detailed explanation: ...
- Case: ...
- Original citation: ...
```

### 2. Outline Version

File: `Learning Roadmap v1 - {Topic}.md` (English) / `学习路线图 v1 - {主题}.md` (中文)

**⚠️ Must preserve the complete three-level structure: H2 → H3 → bullet points**

The outline version must extract **all knowledge points** from the full version; no bullet can be lost:

```markdown
# Learning Roadmap v1 - {Topic Name}

## 01. First Category

### Topic One

- Knowledge Point 1
- Knowledge Point 2
- Knowledge Point 3

### Topic Two

- Knowledge Point 1
- Knowledge Point 2
```

**⚠️ Outline bullet rules**:
- Bullets contain only knowledge point **names/titles**; descriptions, citations, or explanations are forbidden
- Correct: `- Knowledge Point Name`
- Wrong: `- Knowledge Point Name: explanatory text` (contains explanation)
- Wrong: `- Knowledge Point Name [[Source: Document, p.5]]` (contains citation)

**Incorrect example (knowledge points lost, forbidden)**:
```markdown
## 01. First Category

### Topic One

### Topic Two
<!-- ❌ Error: missing bullet points → subsequent file structure creation will lack knowledge points -->
```

## Incremental Mode

When `incremental_mode = true`, the agent adjusts its behavior:

1. **Read the previous roadmap** `Learning Roadmap v1 - {Topic}.md` as context — understand the existing H2/H3/bullet structure
2. **Read only new materials** (files selected by the user in Phase 1.2 incremental file list)
3. **Generate an incremental roadmap** that explicitly marks what is new:

```
📋 Incremental Roadmap

  ## 01. Digital Transformation Fundamentals (existing)
  No changes to existing H3 topics.

  ## 02. Strategic Management (existing)
  ### Supply Chain Resilience (NEW H3 under existing H2)
  - Risk Assessment Methods (NEW)
  - Supplier Diversification Strategy (NEW)

  ## 03. AI in Business (NEW H2)
  ### Machine Learning Fundamentals (NEW)
  - Supervised vs Unsupervised Learning (NEW)
  - Model Evaluation Metrics (NEW)
```

4. Save as `Learning Roadmap v2 - {Topic}.md` (do not overwrite v1)

**Naming convention**:
| Session | File |
|---------|------|
| Initial | `Learning Roadmap v1 - {Topic}.md` |
| 1st incremental | `Learning Roadmap v2 - {Topic}.md` |
| 2nd incremental | `Learning Roadmap v3 - {Topic}.md` |

## Task Steps

### Step 1: Read All Learning Materials

**Strategy: full read first, auto-batch as fallback**

1. Try single-pass full read for each file
2. If it fails, auto-batch (50 pages/batch), auto-continue
3. Record core content summary for each batch
4. Aggregate on completion

**Error handling:**
| Situation | Handling |
|-----------|----------|
| Some pages unreadable | Record skip, continue reading remaining content |
| File corrupted | Inform user, try reading other files |
| Content insufficient for roadmap | Clearly inform user which content is missing |

### Step 2: Generate Full Version Roadmap

Only generate after **confirming all content has been read**.

**🚫 If the source material cannot support at least one H2 containing ≥2 H3 topics, stop and inform the user. A roadmap without H3 topics is structurally invalid — it produces a flat vault with no MOCs, no wikilink scoping, and no topic-level organization.**

**Full version format requirements:**

1. **Detailed knowledge point descriptions** (200+ words)
   - Each knowledge point has detailed explanatory text
   - Enables beginners to understand the concept
   - Includes background, definition, principle, application

2. **Case descriptions**
   - Actual cases from the learning material
   - Explain how the case illustrates the knowledge point

3. **Original citations**
   - Tag document name and page number
   - Briefly quote key original text

4. **source_range annotation** (⚠️ Mandatory — context-extractor.py requires this format)

   **Exact format required by the parser:**
   ```
   **Knowledge Point Title**  `source_range: file.pdf:12-15, 45-48`
   ```
   
   Rules:
   - `source_range:` keyword is backtick-wrapped together with the value: `` `source_range: ...` ``
   - On the SAME line as the bold title (preferred) OR on the very next line
   - Page numbers from actual reading; multiple ranges separated by commas
   - Multi-file: `` `source_range: A.pdf:12-15, B.md` ``
   - Non-PDF files (`.md` / `.txt`): omit page numbers for full text: `` `source_range: overview.md` ``
   
   **⚠️ If source_range annotations are missing, context-extractor.py will fail and Phase 3 will fall back to passing full source files — losing token efficiency.**
   - This annotation is used by `scripts/context-extractor.py` for automation; must not be omitted
   - Single example: `source_range: Digital Transformation Roadmap.pdf:12-15`
   - Multi-range example: `source_range: Platform Strategy.pdf:34-41, 78-82, 156`

5. **H2 Hierarchy Descent Rule** (⚠️ Mandatory)
   - Each H2 (category) **must contain at least 2 H3 topics**
   - If a content block in the material naturally forms only one H3, handle by priority:
     a. **Split**: reorganize the H3's knowledge points into multiple independent H3s (e.g., split "Platform Pricing" + "Platform Governance" out of "Platform Strategy")
     b. **Merge**: merge the content into the semantically closest other H2 (eliminate the original H2, distribute its knowledge points to adjacent H2s)
     c. **Elevate**: remove the H2 level, upgrade H3 to H2 (applicable when the H2 title and sole H3 title are nearly identical)
   - **Forbidden**: redundant hierarchy with only 1 H3 under an H2

### Step 3: Generate Outline Version

Auto-extract from the full version, keeping only:
- H2 titles
- H3 titles
- Unordered list items

### Step 4: Save Two Files

1. Save full version to vault
2. Save outline version to vault
3. Report save results

### Step 5: Parse Statistics from Saved File

**Auto-executed after saving.** The agent re-reads the saved complete roadmap `.md` file and parses its structure — **not model self-reported numbers**.

1. Use the Read tool to open the saved complete roadmap file
2. Parse line by line:
   - Lines starting with `## ` → H2 category count +1
   - Lines starting with `### ` → H3 topic count +1
   - Lines starting with `- ` → knowledge point bullet count +1
3. Report actual file-based statistics:
   - H2 categories, H3 topics, knowledge points
   - Estimated atomic notes count (= bullet count), MOC count (= H3 count)
4. **source_range presence check**: count lines containing `source_range:` in the full roadmap. If zero source_range annotations are found, the context-extractor will fail and Phase 3 will lose token efficiency. This is a ❌ fatal error — go back to Step 2 and regenerate the roadmap with source_range annotations.
5. Run **proportion check**: for each H2 category, compute `knowledge_points ÷ source_pages`:
   - \> 1.0 → ⚠️ over-fragmented, suggest merging adjacent knowledge points
   - 0.1–1.0 → ✅ reasonable
   - \< 0.1 → ⚠️ sparse content, consider merging or expanding

5. Run **hierarchy check**: each H2 must have ≥ 2 H3 topics:
   - 0 → ❌ fatal, H2 must have H3 topics
   - 0 → ❌ fatal, H2 must have H3 topics
   - 1 → ❌ redundant hierarchy, suggest a) split b) merge into adjacent H2 c) elevate H3 to H2
   - ≥ 2 → ✅ reasonable

6. Report findings and ask for confirmation:
   ```
   📊 Roadmap statistics (parsed from saved file):
      H2 categories: 4, H3 topics: 12, Knowledge points: 48
      Estimated atomic notes: 48, MOCs: 12

   ⚠️ Issues found:
      - Strategic Management: 1 H3 → redundant hierarchy
   
   Reply "continue" or "modify + feedback":
   ```

## Constraints

- **Must use content actually present in the learning materials**
- **Do not fabricate or infer non-existent knowledge points**
- Remain objective; do not add viewpoints not in the learning materials
- **Forbidden: generating a roadmap based on partial content**
