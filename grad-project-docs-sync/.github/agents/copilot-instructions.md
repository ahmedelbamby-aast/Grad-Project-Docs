# grad_project Development Guidelines

Auto-generated from all feature plans. Last updated: 2026-02-27

## Active Technologies
- Python 3.11+ (backend), TypeScript ~5.9.3 (frontend) + Django 5.x, Django Channels 4.x, DRF 3.15+, Celery 5.x (backend); React 19.2, Vite 7.3, Zustand 5.x, react-router-dom 7.x, axios (frontend); go2rtc ≥1.9 (video relay); Ultralytics 8.x (YOLO inference) (001-exam-monitor-dashboard)
- PostgreSQL 16 (primary DB, time-partitioned DetectionFrame), Redis 7 (channel layer, cache, Celery broker), local filesystem (video recordings) (001-exam-monitor-dashboard)
- Python 3.10+ (Django 5.1.5) backend, TypeScript ~5.9.3 (React 19.2) frontend + Django 5.1.5, DRF 3.15.2, Django Channels 4.2.2, Celery 5.4.0, Pydantic 2.10.6, React 19.2, Zustand 5.0.11, Axios 1.13.6, Vite 7.3.1 (002-fix-rtsp-camera-feed)
- PostgreSQL 16 (via psycopg 3.2.5), Redis 7 (channels + Celery broker/backend) (002-fix-rtsp-camera-feed)
- Python 3.10+ (backend), TypeScript 5.9.3 (frontend) + Django 5.1.5, DRF 3.15.2, Django Channels 4.2.2, Celery 5.4.0, Pydantic 2.10.6, psycopg 3.2.5, React 19.2, Vite 7.3.1, Zustand 5.0.11, Axios 1.13.6 (002-fix-rtsp-camera-feed)
- PostgreSQL 16 (via `postgres:16-alpine`), Redis 7 (via `redis:7-alpine`) for channel layer + Celery broker (002-fix-rtsp-camera-feed)
- Python 3.11-3.12 (backend), TypeScript 5.9 (frontend) + Django 5.1, DRF, Celery, Redis, Pydantic v2, ONNX/ONNXRuntime/OpenVINO, React 19, Vite 7 (005-architecture-refactoring)
- PostgreSQL + Redis + filesystem model repository + dev-only raw video dataset (005-architecture-refactoring)

## Project Structure

```text
backend/
frontend/
tests/
```

## Commands

cd src; pytest; ruff check .

## Code Style

Python 3.11+ (backend): Ruff linting, mypy strict typing, SOLID principles, ≤30-line functions, custom exceptions, structured logging (no print())
TypeScript ~5.9.3 (frontend): ESLint, strict mode, CSS Modules, Zustand stores separated from components, hooks for DI

## Recent Changes
- 005-architecture-refactoring: Added Python 3.11-3.12 (backend), TypeScript 5.9 (frontend) + Django 5.1, DRF, Celery, Redis, Pydantic v2, ONNX/ONNXRuntime/OpenVINO, React 19, Vite 7
- 002-fix-rtsp-camera-feed: Added Python 3.10+ (backend), TypeScript 5.9.3 (frontend) + Django 5.1.5, DRF 3.15.2, Django Channels 4.2.2, Celery 5.4.0, Pydantic 2.10.6, psycopg 3.2.5, React 19.2, Vite 7.3.1, Zustand 5.0.11, Axios 1.13.6
- 002-fix-rtsp-camera-feed: Added Python 3.10+ (Django 5.1.5) backend, TypeScript ~5.9.3 (React 19.2) frontend + Django 5.1.5, DRF 3.15.2, Django Channels 4.2.2, Celery 5.4.0, Pydantic 2.10.6, React 19.2, Zustand 5.0.11, Axios 1.13.6, Vite 7.3.1


<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
