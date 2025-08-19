# Scraper

A small, no-nonsense Python utility for bulk-downloading protein FASTA sequences by domain/family across multiple databases with a common interface. It’s built for reproducible batch pulls without hand-holding — you get progress bars, rate-limit backoff, and explicit failures instead of silent “success.”

What this does (and doesn’t)

## Does: 

- Searches by domain identifier (e.g., PF00017, cd00184) and optional organism (NCBI TaxID).
- Fetches FASTA for each hit, removes gap characters (-) from sequences, and writes to disk.
- Handles pagination and cursoring per API.
- Implements exponential backoff on 429s; otherwise, it fails fast with a clear error.
- Provides a minimal abstract interface so you can add new databases.

## Doesn’t: 

- Deduplicate across databases (each output file is per-DB).
- Retry forever or mask server failures.
- Overwrite outputs (by design). You must delete or rename existing files.

| DB         | Search semantics                                       | Domain ID example | Organism filter | Notes                                                                                |
| ---------- | ------------------------------------------------------ | ----------------- | --------------- | ------------------------------------------------------------------------------------ |
| UniProt    | `xref:Pfam-<PFAM>` with optional `organism_id:<taxid>` | `PF00017`         | `9606` (human)  | Uses REST `uniprotkb/search` + per-accession FASTA.                                  |
| CDD (NCBI) | `cdXXXXXX` with optional `txid<taxid>[Organism:exp]`   | `cd00184`         | `9606`          | `esearch` → `elink(cdd→protein)` → `efetch(fasta)`. **NCBI API key recommended**.    |
| PDB (RCSB) | Polymer entities annotated with Pfam ID                | `PF00017`         | `9606`          | Searches RCSB text API; downloads entry FASTA; returns **entry IDs** (e.g., `1ABC`). |
