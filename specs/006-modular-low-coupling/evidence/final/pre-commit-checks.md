# Pre-Commit Checks

## Commands

| Command | Result |
| --- | --- |
| `git diff --check` | Passed, line-ending warnings only |
| `manage.py check` | Passed |
| `npm run type-check` | Passed |
| `npm run build` | Passed |
| `npm run lint` | Failed on existing frontend lint errors in `PreviewPlayer.tsx` and `VideoPlayer.tsx` |
| `python scripts\ci\verify_docs_diagrams.py` | Passed |

No backend lint/format/type-check or docs formatter command was found in the task context.
