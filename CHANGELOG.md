# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [1.7.0] - 2026-04-30

### Added
- `MCP_AUTH_TOKEN` env var: when set, every `/mcp` request must include `Authorization: Bearer <token>`; required when binding to `0.0.0.0`
- `MCP_ALLOWED_ORIGINS` env var: comma-separated allowlist of `Origin` headers to mitigate DNS-rebinding; matching is exact normalized-origin (scheme + host + port), not prefix; requests with a disallowed `Origin` receive `403`
- `MCP_ALLOWED_HOSTS` env var: comma-separated allowlist of `Host` header values to mitigate DNS-rebinding (default: `localhost`, `127.0.0.1`, `::1` and the configured bind host); requests with an unrecognized `Host` receive `403`
- `MCP_BODY_LIMIT_BYTES` env var: maximum inbound request body size (default 1 MB); larger bodies receive `413`
- `MCP_REQUEST_TIMEOUT_MS` env var: per-request timeout to prevent slowloris-style attacks (default 30 s)
- `MCP_RATE_LIMIT_DISABLED` env var: set to `"true"` to disable the per-tool fixed-window rate limiter (default: enabled, 60-second windows for add/search/command tools)
- `GITHUB_TOKEN` env var: optional GitHub personal access token used when fetching TRaSH Guides data from `api.github.com`; raises the unauthenticated 60 req/hr rate limit to 5 000 req/hr
- Per-tool rate limiter on destructive/fan-out tools (`*_add_*`, `*_search_*`, `prowlarr_search`, `arr_search_all`, `trash_list_custom_formats`)
- Graceful SIGTERM/SIGINT shutdown: in-flight HTTP requests are drained before the process exits
- Startup duplicate-tool-name assertion: `main()` throws if any tool name is registered more than once, preventing the class of bug fixed in v1.5.4 from recurring silently
- In-flight singleflight coalescing in `TrashCache`: concurrent callers for the same cache key share one outbound fetch instead of fanning out

### Changed
- `ArrClient.request<T>()` promoted from `protected` to `public` to match the de-facto usage by all subclasses
- `getPaginatedQueue` now fetches only the API pages required to satisfy `[offset, offset+limit]` instead of the entire queue; `offset=500, limit=25` now issues exactly one API call

### Fixed
- `fetch` tool returned "not found" for every TRaSH quality profile: the id parser destructured 4 colon-segments into 3 variables, leaving `rawId` as `undefined` (audit finding C3)
- `fetch` tool returned wrong or empty results for *arr items: sonarr and radarr branches used free-text title search on a numeric TVDB/TMDB id instead of a typed `tvdb:`/`tmdb:` lookup (audit finding C4)
- `lidarr_get_quality_profiles` and `lidarr_get_root_folders` were registered twice; the second `TOOLS.push` entries were duplicates of what `addConfigTools('lidarr', …)` already added, and the corresponding `case` blocks in the dispatch switch were unreachable (audit finding C2; recurrence of v1.5.4 pattern)
- Calendar `days: 0` silently fell back to the default 7 or 30 days via falsy coercion (`||`); now `??` is used so 0 correctly returns today-only (audit finding H1)
- Calendar `days: -1` silently produced an invalid date range; now throws `"days must be an integer in [0, 365]"` (audit finding H1)
- Calendar `days: "abc"` threw an uncaught `RangeError` from `Date`; now caught and reported cleanly (audit finding H1)
- `formatBytes(NaN)`, `formatBytes(-1)`, and `formatBytes(undefined)` produced `"NaN undefined"`; now return `"0 B"` (audit finding H2)
- `sonarr_get_series` rendered `"undefined/undefined"` for newly-added series with no statistics; now returns `"unknown"` (audit finding H3)
- `lidarr_get_artists` rendered `"undefined/undefined"` for artists with no statistics; now returns `"unknown"` (audit finding H3)
- HTTP transport now requires `MCP_AUTH_TOKEN` when binding to any wildcard interface (`0.0.0.0`, `::`, `::0`, `*`); startup refuses with a clear error message if the token is absent (audit finding C1)
- Bearer-token comparison is timing-safe and length-leak-resistant (`timingSafeEqualStrings` no longer early-returns on length mismatch); `/health` and `/mcp` use the same comparator
- HTTP error responses no longer leak internal *arr service URLs or raw `error.message` text (audit finding H4)
- All outbound `fetch` calls now carry `AbortSignal.timeout(30_000)` to prevent hung connections from blocking the process indefinitely (audit finding H8)
- `*_URL` env vars are now validated at startup: must use `http://` or `https://`; invalid schemes throw a clear error (audit finding H9)
- Lidarr `getAlbums(artistId=0)` returned the entire library due to JS truthiness (`0` is falsy); fixed with explicit `!== undefined` guard (audit finding M7)
- `mediaServer: "__proto__"` no longer escapes into `Object.prototype` via bracket-notation lookup on `serverMap`; unknown values are rejected with a clean error (codex finding)
- `trash_compare_naming` now uses Sonarr-specific naming keys (`seriesFolderFormat`, `standardEpisodeFormat`) instead of the Radarr keys for the Sonarr branch (audit finding M6)
- Short or purely numeric `search` queries no longer trigger a TRaSH Guides fan-out that could never return useful results (audit finding M4)

### Security
- HTTP transport hardened with bearer auth, `Host` header allowlist, exact-match `Origin` allowlist, body-size cap (Content-Length pre-check + socket-level byte counter), per-request timeout, and graceful drain on shutdown (audit finding C1/H6)
- CI `npm audit --audit-level=high` step is now **blocking** (`continue-on-error: true` removed); previously failing audits were silently ignored (audit finding H10)
- `GITHUB_TOKEN` support allows authenticated GitHub API calls for TRaSH Guides data, reducing exposure to unauthenticated rate-limit exhaustion (audit finding M9)
- Server refuses to start in HTTP mode on `0.0.0.0` without `MCP_AUTH_TOKEN`; anyone on the network would otherwise be able to drive the *arr stack unauthenticated

## [1.6.3] - 2026-04-27

### Fixed
- Fixed remote HTTP MCP mode failing after initialization with `@modelcontextprotocol/sdk` 1.27.x by enabling stateful HTTP sessions with generated MCP session IDs (#5, reported by [@michaelheyman](https://github.com/michaelheyman))

## [1.6.2] - 2026-04-22

### Fixed
- Upgraded `hono` and `path-to-regexp` to resolve 1 high and 2 moderate severity vulnerabilities (cookie name handling, path traversal in `toSSG`, static-serve middleware bypass, JSX SSR injection, IPv6 matching in `ipRestriction`, and ReDoS in `path-to-regexp`)

## [1.6.1] - 2026-04-22

### Added
- Remote HTTP MCP mode via `MCP_TRANSPORT=http` (with `HOST`, `PORT`, `MCP_PATH` env vars)
- Generic `search` and `fetch` tools for hosted/remote MCP clients (e.g. ChatGPT connectors)
- Docker usage examples in the README for both stdio and HTTP mode
- `limit` / `offset` pagination for:
  - `sonarr_get_queue`
  - `radarr_get_queue`
  - `lidarr_get_queue`
- `sonarr_refresh_series` and `radarr_refresh_movie` tools for triggering targeted metadata refresh ([#9](https://github.com/aplaceforallmystuff/mcp-arr/pull/9), contributed by [@ismael9291](https://github.com/ismael9291))
- `limit` / `offset` / `search` pagination for `sonarr_get_series` and `radarr_get_movies` ([#9](https://github.com/aplaceforallmystuff/mcp-arr/pull/9), contributed by [@ismael9291](https://github.com/ismael9291))

### Changed
- The server now starts in TRaSH-only mode even when no local *arr services are configured
- Queue responses now include pagination metadata (`total`, `returned`, `hasMore`, `nextOffset`, etc.)
- Refresh tool responses validate that the target exists before dispatching the command, and echo the resolved `id` / `title` / `year`
- README and server metadata updated to reflect remote MCP support and current versioning

### Fixed
- Broken README architecture image path
- Version drift between `package.json`, `server.json`, and runtime version metadata

### Removed
- Readarr (Books) support — replaced by Booklore + Shelfmark in Docker stack

## [1.5.4] - 2026-03-19

### Fixed
- Duplicate tool registrations for `sonarr_get_quality_profiles`, `sonarr_get_root_folders`, `radarr_get_quality_profiles`, and `radarr_get_root_folders` — each was registered twice (via `addConfigTools()` and manually), causing 8 duplicate entries ([#6](https://github.com/aplaceforallmystuff/mcp-arr/issues/6), reported by [@a1ad](https://github.com/a1ad))
- Updated dependencies to fix 3 high severity vulnerabilities (hono, @hono/node-server, express-rate-limit)

## [1.5.3] - 2026-02-27

### Fixed
- `lidarr_search` now returns `artistName` and `disambiguation` instead of generic `title` field
- `lidarr_search` accepts `term`, `query`, `artist`, or `name` parameters with validation
- Fixed null safety on `overview` field truncation in Lidarr search results

Based on [PR #2](https://github.com/aplaceforallmystuff/mcp-arr/pull/2) by [@bndlfm](https://github.com/bndlfm).

## [1.5.2] - 2026-02-27

### Fixed
- `@modelcontextprotocol/sdk` moved from devDependencies to dependencies — fixes `ERR_MODULE_NOT_FOUND` when installed via `npx` (#3)

## [1.5.1] - 2026-02-27

### Added
- Optional `tags` parameter on all add tools (`sonarr_add_series`, `radarr_add_movie`, `lidarr_add_artist`, `readarr_add_author`) - accepts array of tag IDs from the corresponding `*_get_tags` tool

## [1.5.0] - 2026-02-25

### Added
- `sonarr_add_series` - Add TV series to Sonarr library
- `radarr_add_movie` - Add movies to Radarr library
- `lidarr_add_artist` - Add artists to Lidarr library
- `readarr_add_author` - Add authors to Readarr library
- Helper tools for each service: `*_get_root_folders`, `*_get_quality_profiles`
- `lidarr_get_metadata_profiles` and `readarr_get_metadata_profiles` helpers

### Changed
- Search tool descriptions now reference the add workflow (e.g., "returns tvdbId needed for sonarr_add_series")

### Fixed
- Dependency vulnerabilities in @modelcontextprotocol/sdk, ajv, hono, and qs

## [1.4.1] - 2026-01-13

### Changed
- Updated `@modelcontextprotocol/sdk` to 1.25.2
- Updated `@types/node` to 20.19.29

### Fixed
- Security vulnerability in `qs` dependency (GHSA-6rw7-vpxm-498p)

### Added
- `CLAUDE.md` for Claude Code contributors

## [1.4.0] - 2025-12-01

### Added
- **TRaSH Guides Integration** - Access community-curated quality profiles, custom formats, and naming conventions directly through Claude:
  - `trash_list_profiles` - List available TRaSH quality profiles for Radarr or Sonarr
  - `trash_get_profile` - Get detailed profile with custom formats, scores, and quality settings
  - `trash_list_custom_formats` - List custom formats with optional category filter (hdr, audio, resolution, source, streaming, anime, unwanted, release, language)
  - `trash_get_naming` - Get recommended naming conventions for Plex, Emby, Jellyfin, or standard
  - `trash_get_quality_sizes` - Get recommended min/max/preferred sizes for each quality level
  - `trash_compare_profile` - Compare your profile against TRaSH recommendations
  - `trash_compare_naming` - Compare your naming config against TRaSH recommendations

- New `trash-client.ts` module for fetching and caching TRaSH Guides data from GitHub
- 1-hour cache for TRaSH data to minimize GitHub API calls
- Custom format categorization (HDR, audio, resolution, source, streaming, anime, etc.)

### Purpose
TRaSH Guides tools enable users to reference community best practices for *arr configuration without leaving Claude. Compare your current setup against TRaSH recommendations to identify missing custom formats, quality settings differences, and naming improvements.

## [1.3.0] - 2025-11-29

### Added
- **Configuration Review Tools** - New tools to inspect and analyze *arr service configurations:
  - `{service}_get_quality_profiles` - Detailed quality profile information including allowed qualities, upgrade settings, and custom format scores
  - `{service}_get_health` - Health check warnings and issues detected by the application
  - `{service}_get_root_folders` - Storage paths, free space, and accessibility status
  - `{service}_get_download_clients` - Download client configurations and settings
  - `{service}_get_naming` - File and folder naming conventions
  - `{service}_get_tags` - Tag definitions for content organization
  - `{service}_review_setup` - Comprehensive configuration dump for AI-assisted setup analysis

  These tools are available for Sonarr, Radarr, Lidarr, and Readarr (replace `{service}` with service name).

- New API client methods for configuration retrieval:
  - `getQualityProfiles()` - Full quality profile details
  - `getQualityDefinitions()` - Size limits per quality level
  - `getDownloadClients()` - Download client configurations
  - `getNamingConfig()` - Naming conventions
  - `getMediaManagement()` - File handling settings
  - `getHealth()` - Health check warnings
  - `getTags()` - Tag definitions
  - `getIndexers()` - Per-app indexer configs
  - `getMetadataProfiles()` - Metadata profiles (Lidarr/Readarr only)

### Purpose
The new configuration review tools enable natural language conversations about *arr setup optimization. Users can ask Claude to review their configuration and suggest improvements, especially helpful for understanding complex quality profiles and media management settings.

## [1.2.0] - 2025-11-28

### Added
- Sonarr episode management tools:
  - `sonarr_get_episodes` - List episodes for a series with availability status
  - `sonarr_search_missing` - Trigger search for missing episodes
  - `sonarr_search_episode` - Search for specific episodes
- Radarr download tools:
  - `radarr_search_movie` - Trigger search for a movie
- Lidarr album management tools:
  - `lidarr_get_albums` - List albums for an artist with availability status
  - `lidarr_search_album` - Trigger search for a specific album
  - `lidarr_search_missing` - Search for all missing albums for an artist
  - `lidarr_get_calendar` - View upcoming album releases
- Readarr book management tools:
  - `readarr_get_books` - List books for an author
  - `readarr_search_book` - Trigger search for specific books
  - `readarr_search_missing` - Search for missing books
  - `readarr_get_calendar` - View upcoming book releases
- Prowlarr indexer tools:
  - `prowlarr_test_indexers` - Health check all indexers
  - `prowlarr_get_stats` - Indexer statistics

## [1.1.0] - 2025-11-28

### Fixed
- Corrected API version for Lidarr, Readarr, and Prowlarr (use `/api/v1` instead of `/api/v3`)
- Added configurable `apiVersion` property to base ArrClient class

### Added
- `server.json` for MCP registry compatibility

## [1.0.0] - 2025-11-28

### Added
- Initial release with MCP tools for *arr media management suite
- **Sonarr** (TV) tools:
  - `sonarr_get_series` - List all TV series in library
  - `sonarr_search` - Search for TV series to add
  - `sonarr_get_queue` - View download queue
  - `sonarr_get_calendar` - View upcoming episodes
- **Radarr** (Movies) tools:
  - `radarr_get_movies` - List all movies in library
  - `radarr_search` - Search for movies to add
  - `radarr_get_queue` - View download queue
  - `radarr_get_calendar` - View upcoming releases
- **Lidarr** (Music) tools:
  - `lidarr_get_artists` - List all artists in library
  - `lidarr_search` - Search for artists to add
  - `lidarr_get_queue` - View download queue
- **Readarr** (Books) tools:
  - `readarr_get_authors` - List all authors in library
  - `readarr_search` - Search for authors to add
  - `readarr_get_queue` - View download queue
- **Prowlarr** (Indexers) tools:
  - `prowlarr_get_indexers` - List configured indexers
  - `prowlarr_search` - Search across all indexers
- **Cross-service** tools:
  - `arr_status` - Check health of all configured services
  - `arr_search_all` - Search across all media types
