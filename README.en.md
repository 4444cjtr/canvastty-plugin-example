# canvastty-plugin-example

Example of a working plugin for plugin indexing.

---

## CanvasTTY plugin indexing module

This repository is the reference structure of a plugin for the CanvasTTY plugin indexing
system. Below is how indexing works: what is searched, how it is validated, what limits
apply, and why.

### What plugin indexing is

CanvasTTY discovers plugins not through a registry but by **indexing GitHub repositories**:

1. **Repository search** — GitHub search by name with the `canvastty-plugin-` prefix
   (`canvastty-plugin-* in:name`). Only public, anonymously accessible repositories are found.
2. **Manifest read** — for every found repository the manifest is read from
   `metadata/canvastty.plugin.json` (the legacy root `canvastty.plugin.json` is also supported).
3. **Validation** — the manifest passes strict validation (see "Limits").
4. **Platform filtering** — a repository appears in the showcase only if it declares a
   compatible `platforms` entry.
5. **Display** — installed plugins are excluded from the showcase; the remaining ones show
   version, author, description (truncated), icon, and the host version the plugin was
   written for.

Installation happens from the same repository: the repo archive (`main` branch) is
downloaded, the manifest is validated again, and files are laid out in
`userData/plugins/<pluginId>/`.

### Plugin structure (`metadata/` standard)

```
canvastty-plugin-<name>/
├── metadata/
│   ├── canvastty.plugin.json   ← manifest (required)
│   └── icon.png                ← 128×128 icon (required for the showcase)
├── app.html                    ← contribution entry files (relative paths)
├── sample.gif                  ← any plugin assets
└── README.md
```

**Why `metadata/`**: all metadata (manifest, icon) lives in one folder and never mixes with
plugin code. The indexer looks only there — plugin code can change without reindexing.

### Manifest

```json
{
  "apiVersion": 1,
  "id": "com.example.plugin",
  "name": "example",
  "version": "1.0.0",
  "description": "example of a working plugin for plugin indexing",
  "author": "4444cjtr",
  "homepage": "https://github.com/4444cjtr/canvastty-plugin-example",
  "permissions": [],
  "platforms": ["canvastty"],
  "minHostVersion": "1.2.1",
  "contributions": [
    {
      "id": "example-gif",
      "kind": "canvas-app",
      "title": "Example",
      "description": "Shows an example GIF in the workspace.",
      "entry": "app.html",
      "defaultSize": { "width": 480, "height": 320 },
      "minSize": { "width": 320, "height": 220 }
    }
  ]
}
```

Fields:

| Field | Required | Purpose |
|---|---|---|
| `apiVersion` | yes | Manifest format version. Currently `1`. Breaking changes bump the number. |
| `id` | yes | Unique identifier (reverse domain, ≤ 80 chars). Determines the install folder and the showcase/registry key. |
| `name` | yes | Human-readable name (≤ 80 chars). |
| `version` | yes | Semver plugin version (`X.Y.Z`). Used for update checks: if the repo version is higher than the installed one, an update is offered. |
| `description` | yes | Description (≤ 2 000 chars). Truncated to 400 with an ellipsis in the UI. |
| `author` | no | Author nickname (≤ 120 chars). Shown in the installed card and the showcase. |
| `homepage` | no | Repository/site link. |
| `permissions` | yes | Array of permissions (only from the known set). Requested from the user at install time. |
| `platforms` | no | Array of platforms the plugin targets, e.g. `["canvastty"]`. **Showcase filter**: if declared and does not include `canvastty`, the plugin is not indexed. Multi-platform plugins (`["canvastty", "canvastty-superkruto"]`) are visible. Absent field = legacy plugin, compatible with everything. |
| `minHostVersion` | no | Minimum CanvasTTY version the plugin is written for (semver). Shown as `CanvasTTY:X.Y.Z` in the showcase/cards: green if the plugin targets a newer host, orange if an older one. **Does not block** installation — informational only. |
| `contributions` | yes | 1–32 entry points (see below). |
| `settingsContribution` | no | id of a `canvas-app` contribution opened as the plugin's settings screen. |
| `modules` | no | Modules (optional code packages with their own permissions); `coreFiles` is required when modules exist. |

### Contributions (entry points)

| kind | Purpose | Size |
|---|---|---|
| `canvas-app` | A node in the workspace (like this plugin's GIF). | `defaultSize`/`minSize`: width ≥ 320, height ≥ 220 |
| `home-widget` | A widget on the Home screen. | `defaultSize` in columns/rows: 1–48 × 1–36 |
| `window` | A separate window. | `defaultSize`/`minSize`: width ≥ 320, height ≥ 220 |

### Limits

#### Manifest validation

| What | Limit | Why |
|---|---|---|
| `id` | ≤ 80 chars | Registry key, on-disk folder, protocol links. |
| `name` | ≤ 80 chars | Card title. |
| `version` | ≤ 40 chars, semver | Update comparison. |
| `description` | ≤ 2 000 chars | The validator accepts long descriptions; the UI truncates to **400** with `…` — a balance between completeness in the repo and readability in the interface. |
| `author` | ≤ 120 chars | The "Author: …" line. |
| `icon` path | ≤ 180 chars | Icon file path. |
| `permissions` | known set only | Security: a plugin cannot request an unknown permission. |
| `contributions` | 1–32 | Protection against "sheet-like" manifests. |
| `settingsContribution` | ≤ 64 chars, must reference a canvas-app | Cannot point to a widget. |

#### Package (repository archive)

| What | Limit | Why |
|---|---|---|
| Archive size | ≤ 25 MB | Download bound. |
| File count | ≤ 500 | Protection against zip bombs and junk files. |
| Single asset size | ≤ 8 MB | Individual files (GIFs, media). |
| Manifest | ≤ 128 KB | Metadata must not be heavy. |
| Icon | ≤ 512 KB, 128×128 | Light batch loading of showcase icons. |
| `storage` (per plugin) | ≤ 64 KB | Plugin storage quota. |

#### Indexing and search

| What | Limit | Why |
|---|---|---|
| Prefix search | `canvastty-plugin-*` | Only targeted repositories, no noise. |
| Search results | ≤ 10 | Fast UI response. |
| Showcase | ≤ 10 per page (pagination) | Readability. |
| Installed | excluded from the showcase | Do not offer what is already installed. |
| GitHub timeout | 15 s | Do not hang the UI on a slow network. |

### Rate-limit optimizations

The indexer is careful with the GitHub API:

- **Repository metadata cache** (`githubMetadataCache`) — owner/branch are cached per session.
- **GraphQL batching** — manifests and icons are requested in batches (up to 8 repos per
  request) in a single call to `api.github.com/graphql` instead of one-by-one.
- **Tokenless fallback** — without a token, files are read from `raw.githubusercontent.com`.
- **Icon batching** — `plugins:icon` accepts an array of URLs and returns
  `Record<url, dataUrl|null>`; the UI never issues one request per icon.

### Host version in the manifest

`minHostVersion` is **data, not a restriction**:

- The showcase and cards show `CanvasTTY:X.Y.Z` under the author.
- Green — the plugin is written for a newer host than installed (informational; update the
  host if you wish).
- Orange — the plugin is written for an older host.
- Installation/updates are **not blocked** by a version mismatch.
