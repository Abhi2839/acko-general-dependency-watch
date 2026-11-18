# Dependency Watch — Progress

**Updated:** 18 Nov 2026 · **Status:** core engine working, baseline dashboard shipped; enterprise dashboard, history store, and update proposals are next.

Related: [`PLAN.md`](PLAN.md) · [`DESIGN.md`](DESIGN.md) · [`watcher/README.md`](watcher/README.md)

## Built so far

| Area | State | Evidence |
| --- | --- | --- |
| Service inventory (8 services) | ✅ | `watcher/collect.py` — `service.yaml` metadata, tier/owner/exposure |
| Dependency parsing — npm | ✅ | `package.json` + lockfile v3: resolved versions, direct vs transitive, prod vs dev |
| Dependency parsing — Python | ✅ | `requirements.txt`, pip-compile `requirements.lock` with `# via` chains, `requirements-dev.txt` |
| Unknown-version handling | ✅ | Unpinned services (docs-gen-svc) surface `version: null`, `version_source: unknown` |
| OSV feed ingestion | ✅ | Fetches `/v1/advisories`, validates shape, handles add/change/withdraw |
| Version ↔ range matching | ✅ | SEMVER (npm) and ECOSYSTEM (PyPI); boundaries and multiple ranges covered by tests |
| Risk prioritisation | ✅ | P0/P1/P2 from severity + tier + internet exposure, with human-readable reason |
| Persistent lifecycle | ✅ | OPEN → RESOLVED → REOPENED; stable `DW-F-000001` references; dedupe by `(service, package, advisory)` |
| Audit history | ✅ | Every NEW/RESOLVED/REOPENED event with timestamp and duration |
| Safety rules | ✅ | Failed scan/feed change never marks a finding fixed; unknown version never resolves; withdrawn ≠ fix |
| Fix-time estimate | ✅ | Median of ≥3 dependency-change resolutions per priority/tier; no estimate without samples |
| API | ✅ | `GET /api/dashboard`, `POST /api/scan`, `GET /health`, background scan loop |
| Offline demo fallback | ✅ | Unreachable live feed uses the kit's bundled `feed-1` advisories, labelled in the UI/API; `FEED_MODE=live` disables, `FEED_MODE=bundled` forces it |
| Health observability | ✅ | `/health` separates `scan_status` from `feed_status` (`live`/`fallback`/`unavailable`), maps overall `starting`/`ok`/`degraded`, and never triggers a scan |
| Baseline dashboard | ✅ | Findings, services, vulnerabilities tabs; search/filter; scan button; live polling |
| Dashboard polish | ✅ | Health pills (overall/feed/scan), severity + sort filters, clickable severity cards, service risk summaries, advisory ranges/fixed versions, open rows preserved across refresh |
| Tests | ✅ | 9 tests passing (`python -m unittest discover -s watcher/tests -v`) |
| Live check | ✅ | Feed 1: 14 findings; switch to feed 2: 16 findings including DW-2026-0016/0017 |

## Not built yet

| Item | Notes |
| --- | --- |
| Enterprise UI (sidebar, tables, charts, history view, impact graph, investigation pages) | Planned in [`PLAN.md`](PLAN.md) §7; currently one page with three tabs |
| History storage additions | `last_detected_at`, `version_observations`, `advisory_snapshots` — PLAN §12.1 |
| Manual solution records | Agreed dashboard feature: append-only department entries (RCA, applied version, author, timestamp) with API + Investigation form — PLAN §12.2 |
| Analysis enrichment (CVSS, exploit method, root cause) | Not present in the mock feed; the UI shows "Not available" |
| Impact beyond package chains | No service-to-service or business-workflow data in the kit |
| Update automation | Proposal-first only; no repository access; `services/` fixtures are simulated with `scripts/change.sh` |
| Notifications, developer assignment | Explicitly extra (PLAN §10) |
| SLA / "deadline" | Not agreed; estimates only, clearly labelled |

## Known limitations

- Lockfiles show **resolved** versions, not what is deployed. Results are evidence, not proof of production state.
- Solution records are self-declared by the department: there is no authentication in the kit, and a record is not proof that the fix reached production — the rescan is what closes a finding.
- docs-gen-svc has no lockfile, so its versions are unknown and are never matched definitively.
- A dependency change is recorded as the resolution reason for fix-time estimates; withdrawn advisories and advisory-range changes are excluded from the estimate cohort.
- Test history is seeded by scans and `change.sh` runs; real-world history starts accumulating only once the app scans continuously.
- The dashboard is local-only and has no authentication.
- Offline snapshots come from the kit's bundled fixtures, not the live feed; they are labelled and can be stale relative to the feed server.

## How to verify

```sh
docker compose up --build                                  # kit feed (+ webhook receiver)
python -m pip install -r watcher/requirements.txt
python -m unittest discover -s watcher/tests -v            # 9 passing
python -m uvicorn watcher.app:app --port 8000              # http://localhost:8000
```

Useful demo steps:

1. Load the dashboard; confirm 8 services and feed-1 findings.
2. `curl -X POST localhost:7100/control/feed/2` → next scan shows the feed-2 additions.
3. `bash scripts/change.sh fix-quote-axios` → scan resolves the quote-svc axios finding.
4. `bash scripts/change.sh reset` → the same finding reopens with its reference intact.

## Next milestones

1. Database migration + tests (history fields, snapshots, version observations, `solutions`).
2. API fields for current/historical views + solution append endpoint.
3. New UI: overview, current vulnerabilities, investigation (with the manual solution form).
4. History and resolution-time views.
5. Impact graph and analytics.
6. Recommendation-only update proposals, demonstrated end-to-end with `change.sh`; the solution entry stays manual.
