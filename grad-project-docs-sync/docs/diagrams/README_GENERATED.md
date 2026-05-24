# Generated Diagrams

This folder contains compiler-generated Mermaid documentation and source graph data.

## CI Verification

Run:

```powershell
.\.venv\Scripts\python.exe scripts\ci\verify_generated_diagrams.py
```

The verifier:

- Runs `scripts/diagrams/build_all.py --check`.
- Fails on drift between generated output and committed files.
- Fails if required generated artifacts are missing (`.mmd`, `.md`, and `data/*.json`).

## Expected Structure

- `docs/diagrams/generated/**/*.mmd`
- `docs/diagrams/generated/**/*.md`
- `docs/diagrams/generated/data/**/*.json`
