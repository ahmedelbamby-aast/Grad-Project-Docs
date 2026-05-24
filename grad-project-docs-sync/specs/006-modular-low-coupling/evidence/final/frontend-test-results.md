# Frontend Test Results

## Commands

| Command | Result |
| --- | --- |
| `npm test -- --run` | Passed: 28 files, 202 tests |
| `npm run type-check` | Passed |
| `npm run build` | Passed after adapter type fixes |
| `npm run test:e2e -- full-baseline` | Passed: 1 test; Vite emitted WebSocket ECONNABORTED logs with zero exit status |
| `npm run lint` | Failed on existing lint errors in `PreviewPlayer.tsx` and `VideoPlayer.tsx`, plus hook warnings |
| `npm test -- --coverage --run` | Tests passed, coverage threshold failed: 62.99% statements, 54.96% branches, 64.4% functions, 64.14% lines |
