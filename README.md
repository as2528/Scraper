# Final Scraper — README

A small, **no-nonsense** Python utility for bulk-downloading protein **FASTA** sequences by **domain/family** across multiple databases with a common interface. It’s built for reproducible batch pulls without hand-holding — you get progress bars, rate-limit backoff, and explicit failures instead of silent “success.”

---

## What this does (and doesn’t)

**Does**

* Searches by **domain identifier** (e.g., `PF00017`, `cd00184`) and optional **organism** (NCBI TaxID).
* Fetches **FASTA** for each hit, removes gap characters (`-`) from sequences, and writes to disk.
* Handles **pagination** and **cursoring** per API.
* Implements **exponential backoff** on 429s; otherwise, it fails fast with a clear error.
* Provides a minimal **abstract interface** so you can add new databases.

**Doesn’t**

* Deduplicate across databases (each output file is per-DB).
* Retry forever or mask server failures.
* Overwrite outputs (by design). You must delete or rename existing files.

---

## Supported databases

| DB         | Search semantics                                       | Domain ID example | Organism filter | Notes                                                                                |
| ---------- | ------------------------------------------------------ | ----------------- | --------------- | ------------------------------------------------------------------------------------ |
| UniProt    | `xref:Pfam-<PFAM>` with optional `organism_id:<taxid>` | `PF00017`         | `9606` (human)  | Uses REST `uniprotkb/search` + per-accession FASTA.                                  |
| CDD (NCBI) | `cdXXXXXX` with optional `txid<taxid>[Organism:exp]`   | `cd00184`         | `9606`          | `esearch` → `elink(cdd→protein)` → `efetch(fasta)`. **NCBI API key recommended**.    |
| PDB (RCSB) | Polymer entities annotated with Pfam ID                | `PF00017`         | `9606`          | Searches RCSB text API; downloads entry FASTA; returns **entry IDs** (e.g., `1ABC`). |

---

## Requirements

* Python **3.9+**
* `requests`, `tqdm`

```bash
python -m venv .venv
source .venv/bin/activate    # Windows: .venv\Scripts\activate
pip install requests tqdm
```

---

## Quick start

The script is designed to be run as a module with parameters hardcoded in `__main__` (see below), or by calling `run_pipeline(...)` from your own code.

### Example: run all three DBs (as in `__main__`)

```python
if __name__ == "__main__":
    for db in ("uniprot", "cdd", "pdb"):
        run_pipeline(
            db_name=db,
            domain="PF00017" if db != "cdd" else "cd00184",
            organism_id=9606,
            out_file=f"{db}.fasta",
            max_proteins=700
        )
```

**Behavior**

* Writes `uniprot.fasta`, `cdd.fasta`, `pdb.fasta` in the working directory.
* **Will not overwrite** existing files; raises `FileExistsError` if a target exists.
* Shows a `tqdm` progress bar per database.
* Sleeps \~0.34s between FASTA requests (tunable) and backs off on 429s.

---

## Programmatic usage

### High-level orchestration

```python
run_pipeline(
    db_name: str,             # "uniprot" | "cdd" | "pdb"
    domain: str,              # e.g. "PF00017" or "cd00184"
    organism_id: int | None,  # NCBI TaxID, or None for all organisms
    out_file: str,            # output FASTA filename
    max_proteins: int | None  # stop after this many hits; None = all
)
```

**Guarantees**

* Refuses to overwrite `out_file`.
* Raises on network/API errors rather than silently skipping.

### Lower-level batch fetch

```python
fetch_domain_proteins_fasta(
    db: ProteinDatabase,   # instance: UniProtDatabase() | CDDDatabase() | PDBDatabase()
    domain: str,
    organism_id: int | None,
    out_file: str,
    max_proteins: int | None,
    retry_attempts: int,   # e.g., 3
    pause: float           # seconds between accessions, e.g., 0.34
)
```

---

## Architecture

```text
ProteinDatabase (ABC)
 ├─ search_accessions(domain, organism_id?, max_proteins?) -> list[str]
 └─ fetch_fasta(accession) -> str (FASTA text, with '-' stripped)

Implementations:
 ├─ UniProtDatabase
 ├─ CDDDatabase
 └─ PDBDatabase
```

* **Why an ABC?** Databases vary wildly (query languages, pagination, mapping steps). The ABC enforces the minimum surface area to plug in a new source without editing the orchestrator.

---

## Implementation details per DB

### UniProt

* **Search:** `GET /uniprotkb/search` with `query=xref:Pfam-<PFAM>` and optional `organism_id:<taxid>`.
* **Pagination:** `nextCursor`.
* **Fetch:** per-accession `GET /uniprotkb/<ACC>.fasta`.
* **Cleaning:** joins wrapped lines; removes `-`.

### CDD (NCBI E-utilities)

* **Search:** `esearch.fcgi?db=cdd&term=<domain [+ organism]>`.
* **Map:** `elink.fcgi?dbfrom=cdd&db=protein&id=...` (chunks of 500).
* **Fetch:** `efetch.fcgi?db=protein&id=<GI or ACC>&rettype=fasta`.
* **Rate limiting:** include `tool`, `email`, and **add `api_key`** via `NCBI_EXTRAS` for higher throughput.
* **Reality check:** It’s normal to see **no protein links** for some CDD terms + organism constraints; that’s an upstream data situation, not your code.

### PDB (RCSB)

* **Search:** POST `/rcsbsearch/v2/query` for `rcsb_polymer_entity_annotation.annotation_id == <PFAM>`, optionally AND organism taxonomy lineage id.
* **Return:** polymer **entity identifiers** like `1ABC_1`; code deduplicates to **entry IDs** (`1ABC`).
* **Fetch:** `https://www.rcsb.org/fasta/entry/<ENTRY_ID>`.
* **Filtering:** The query already restricts by organism, so post-filtering of chains is generally unnecessary.

---

## Output format

* A single multi-record FASTA file per database.
* Each record header is whatever the source provides; sequences are **gap-stripped** (no `-`).
* No deduplication across databases; if you want cross-DB merges, do that downstream.

---

## Error model (intended and explicit)

* **HTTP not 200** → raises `RuntimeError` with context.
* **No hits** after pagination/mapping → raises `RuntimeError("No hits for …")`.
* **Target `out_file` exists** → raises `FileExistsError`.
* **429 (rate limit)** in `fetch_fasta` → **exponential backoff** up to `retry_attempts`, then raise.

This is deliberate. Silent fallbacks hide bad data pulls.

---

## Tuning & etiquette

* Adjust `pause` (default `0.34s`) based on the API’s usage policy and your API key status.
* For NCBI, set your **email** (done) and consider adding an **`api_key`** in `NCBI_EXTRAS`:

  ```python
  NCBI_EXTRAS = {"tool": "ProteinBatchDownloader", "email": "you@lab.org", "api_key": "YOUR_KEY"}
  ```
* For very large pulls, also lower `max_proteins` and run per-DB to isolate failures.

---

## Extending to a new database

1. Subclass `ProteinDatabase`.
2. Implement:

   * `search_accessions(domain, organism_id=None, max_proteins=None) -> list[str]`
   * `fetch_fasta(accession) -> str` (return a **clean** FASTA record; remove `-`).
3. Register it in `run_pipeline`’s DB map:

   ```python
   DB_MAP = {"uniprot": UniProtDatabase, "cdd": CDDDatabase, "pdb": PDBDatabase, "mydb": MyDatabase}
   ```

Keep it deterministic: **no hidden global state**.

---

## Reproducibility checklist

* Code pins the APIs by their current base URLs; if an API moves, this will break loudly.
* No on-disk caching; if you need that, wrap `fetch_fasta` with a simple file cache keyed by accession.
* ENV-free by default; NCBI key is optional.

---

## Troubleshooting

### “`RuntimeError: No hits for cd00184 (organism=9606)`”

* Common with **CDD → protein** mapping when organism constraints are tight or when that conserved domain has no curated protein links for the taxon.
* Sanity checks:

  * Drop `organism_id` and see if protein links exist at all.
  * Reduce `max_proteins`.
  * Provide an **NCBI `api_key`** to avoid silent throttling affecting `elink`.
  * Try a different CDD or the Pfam equivalent on UniProt to verify the domain itself is populated.

### “`429 Too Many Requests`”

* Increase `pause`.
* Increase `retry_attempts`.
* Add the relevant API key (NCBI) or slow down.

### PDB gives very few entries

* That’s expected for some Pfams + organism filters. The RCSB search is entity-level and strict.

---

## Example: selective pulls

```python
# Pull 500 human SH3-domain proteins from UniProt
run_pipeline("uniprot", domain="PF00018", organism_id=9606, out_file="uniprot_sh3.fasta", max_proteins=500)

# Pull all proteins linked to cd00184 across all organisms (may be sparse)
run_pipeline("cdd", domain="cd00184", organism_id=None, out_file="cdd_cd00184.fasta", max_proteins=None)

# Pull up to 200 PDB entries annotated with PF00017 (human)
run_pipeline("pdb", domain="PF00017", organism_id=9606, out_file="pdb_pf00017_hs.fasta", max_proteins=200)
```

---

## Code quality notes (by design)

* **Explicit failures** over “best-effort.” If something upstream is empty or broken, you should know immediately.
* **No overwrites**: safer for batch jobs and audits.
* **Sequence cleaning** is minimal (only gap `-` removal). If you want to normalize headers or strip whitespace, do it post-hoc.

---

## License & attribution

Choose whatever your lab uses (MIT, BSD-3, Apache-2.0). If you’re hitting NCBI, comply with their usage policies (email/tool name, key for higher rate).

---

