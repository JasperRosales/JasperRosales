# Journal App — Agent Development Guide

> **Scope:** Production-ready, cross-platform journal application.  
> **Platforms:** Linux · Windows · Android (phone & tablet)  
> **Database:** Built-in SQLite (embedded — no external server required)

---

## Table of Contents

1. [Project Overview](#1-project-overview)  
2. [Tech Stack](#2-tech-stack)  
3. [Repository Structure](#3-repository-structure)  
4. [Architecture](#4-architecture)  
5. [Database Schema](#5-database-schema)  
6. [Feature Set](#6-feature-set)  
7. [API / IPC Contract](#7-api--ipc-contract)  
8. [UI/UX Guidelines](#8-uiux-guidelines)  
9. [Security](#9-security)  
10. [Testing Strategy](#10-testing-strategy)  
11. [CI/CD Pipeline](#11-cicd-pipeline)  
12. [Build & Release](#12-build--release)  
13. [Coding Conventions](#13-coding-conventions)  
14. [Contribution Workflow](#14-contribution-workflow)  

---

## 1. Project Overview

**Journal App** is a private, offline-first journaling application.  
Users can create, edit, delete, tag, and search personal journal entries.  
All data is stored locally in an encrypted SQLite database — nothing leaves the device without explicit user action (export / backup).

### Goals

| Goal | Detail |
|------|--------|
| **Cross-platform** | Single codebase runs on Linux, Windows, and Android (phone + tablet) |
| **Offline-first** | Full functionality with no internet connection |
| **Privacy-first** | AES-256 database encryption; no telemetry by default |
| **Production-ready** | Automated tests, CI/CD, signed releases, semantic versioning |

---

## 2. Tech Stack

| Layer | Choice | Reason |
|-------|--------|--------|
| **App shell** | [Tauri v2](https://tauri.app/) (Rust core) | Single codebase → Linux, Windows, Android; tiny bundle size |
| **UI framework** | React 18 + TypeScript | Familiar ecosystem; component reuse across platforms |
| **Styling** | Tailwind CSS + shadcn/ui | Responsive, utility-first; great tablet adaptation |
| **State management** | Zustand | Lightweight; easy persistence via middleware |
| **Database** | SQLite via `tauri-plugin-sql` | Embedded, zero-config, battle-tested |
| **ORM / query builder** | [Kysely](https://kysely.dev/) with SQLite dialect | Type-safe SQL query builder; works with the Tauri SQL plugin |
| **Encryption** | `sqlcipher` (via Tauri SQL plugin build flag) | AES-256 at rest |
| **Rich text editor** | [Tiptap](https://tiptap.dev/) | Extensible, headless, no runtime deps |
| **Search** | FTS5 (SQLite full-text search) | Built-in; no extra service |
| **Testing — unit** | Vitest + React Testing Library | Fast, ESM-native |
| **Testing — e2e** | Playwright (desktop) + Detox (Android) | Platform-appropriate |
| **Linting** | ESLint (typescript-eslint) + Prettier | Consistent style |
| **CI/CD** | GitHub Actions | Native to this repo |
| **Release signing** | Tauri Updater + platform keystores | Secure auto-updates |

> **Go backend note:** For future cloud-sync or export-to-server features, a Go (Gin + GORM) microservice can be added without modifying the desktop/mobile core.

---

## 3. Repository Structure

```
journal-app/
├── src-tauri/                  # Rust / Tauri core
│   ├── src/
│   │   ├── main.rs             # Tauri entry point
│   │   ├── commands/           # IPC command handlers
│   │   │   ├── entries.rs
│   │   │   ├── tags.rs
│   │   │   └── settings.rs
│   │   ├── db/
│   │   │   ├── migrations/     # SQL migration files (versioned)
│   │   │   └── mod.rs
│   │   └── crypto.rs           # Passphrase key derivation
│   ├── Cargo.toml
│   └── tauri.conf.json
│
├── src/                        # React / TypeScript frontend
│   ├── components/
│   │   ├── editor/             # Tiptap rich-text editor wrapper
│   │   ├── layout/             # Sidebar, TopBar, MobileNav
│   │   ├── entries/            # EntryCard, EntryList, EntryDetail
│   │   ├── tags/               # TagPicker, TagBadge
│   │   └── ui/                 # shadcn/ui re-exports
│   ├── hooks/                  # Custom React hooks
│   ├── store/                  # Zustand stores
│   ├── services/               # IPC wrappers (calls Tauri commands)
│   ├── types/                  # Shared TypeScript interfaces
│   ├── utils/
│   ├── App.tsx
│   └── main.tsx
│
├── tests/
│   ├── unit/                   # Vitest unit tests
│   ├── integration/            # Vitest integration (mocked IPC)
│   └── e2e/                    # Playwright scripts
│
├── .github/
│   └── workflows/
│       ├── ci.yml              # Lint + test on every PR
│       └── release.yml         # Build & publish on version tag
│
├── tailwind.config.ts
├── vite.config.ts
├── tsconfig.json
├── package.json
└── agent.md                    # ← this file
```

---

## 4. Architecture

```
┌──────────────────────────────────────────────────────────┐
│                     Frontend (React/TS)                   │
│  components ─► stores (Zustand) ─► services (IPC calls)  │
└────────────────────────┬─────────────────────────────────┘
                         │  Tauri IPC (invoke / event)
┌────────────────────────▼─────────────────────────────────┐
│                  Tauri Core (Rust)                        │
│  Commands → DB layer → SQLite (tauri-plugin-sql)          │
│           → Crypto (argon2 + AES-256 via SQLCipher)       │
└──────────────────────────────────────────────────────────┘
                         │
                  SQLite file on disk
            (platform app-data directory)
```

### Platform Data Paths

| Platform | Default path |
|----------|-------------|
| Linux | `~/.local/share/journal-app/journal.db` |
| Windows | `%APPDATA%\journal-app\journal.db` |
| Android | `/data/data/com.journal_app/databases/journal.db` |

Tauri resolves these automatically via `tauri::api::path::app_data_dir()`.

---

## 5. Database Schema

All migrations live in `src-tauri/src/db/migrations/` and are run automatically at startup.

### Migration 001 — Core tables

```sql
-- migrations/001_init.sql

CREATE TABLE IF NOT EXISTS settings (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS entries (
    id           TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
    title        TEXT NOT NULL DEFAULT '',
    body         TEXT NOT NULL DEFAULT '',       -- Tiptap JSON (stringified)
    body_text    TEXT NOT NULL DEFAULT '',       -- plain-text copy for FTS
    mood         TEXT,                           -- e.g. "happy", "neutral", "sad"
    weather      TEXT,                           -- optional metadata
    created_at   INTEGER NOT NULL DEFAULT (strftime('%s', 'now')),
    updated_at   INTEGER NOT NULL DEFAULT (strftime('%s', 'now')),
    deleted_at   INTEGER                         -- soft delete
);

CREATE TABLE IF NOT EXISTS tags (
    id    TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
    name  TEXT NOT NULL UNIQUE COLLATE NOCASE,
    color TEXT NOT NULL DEFAULT '#6B7280'
);

CREATE TABLE IF NOT EXISTS entry_tags (
    entry_id TEXT NOT NULL REFERENCES entries(id) ON DELETE CASCADE,
    tag_id   TEXT NOT NULL REFERENCES tags(id)    ON DELETE CASCADE,
    PRIMARY KEY (entry_id, tag_id)
);

-- Full-text search virtual table
CREATE VIRTUAL TABLE IF NOT EXISTS entries_fts
    USING fts5(title, body_text, content='entries', content_rowid='rowid');

-- Keep FTS in sync
CREATE TRIGGER IF NOT EXISTS entries_fts_insert
    AFTER INSERT ON entries BEGIN
        INSERT INTO entries_fts(rowid, title, body_text)
        VALUES (new.rowid, new.title, new.body_text);
    END;

CREATE TRIGGER IF NOT EXISTS entries_fts_update
    AFTER UPDATE ON entries BEGIN
        INSERT INTO entries_fts(entries_fts, rowid, title, body_text)
        VALUES ('delete', old.rowid, old.title, old.body_text);
        INSERT INTO entries_fts(rowid, title, body_text)
        VALUES (new.rowid, new.title, new.body_text);
    END;

CREATE TRIGGER IF NOT EXISTS entries_fts_delete
    AFTER DELETE ON entries BEGIN
        INSERT INTO entries_fts(entries_fts, rowid, title, body_text)
        VALUES ('delete', old.rowid, old.title, old.body_text);
    END;

-- Auto-update updated_at
CREATE TRIGGER IF NOT EXISTS entries_updated_at
    AFTER UPDATE ON entries BEGIN
        UPDATE entries SET updated_at = strftime('%s', 'now')
        WHERE id = new.id;
    END;
```

### Migration 002 — Attachments (images)

```sql
-- migrations/002_attachments.sql

CREATE TABLE IF NOT EXISTS attachments (
    id        TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
    entry_id  TEXT NOT NULL REFERENCES entries(id) ON DELETE CASCADE,
    filename  TEXT NOT NULL,
    mime_type TEXT NOT NULL,
    size      INTEGER NOT NULL,
    created_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now'))
);
```

---

## 6. Feature Set

### MVP (v1.0)

- [ ] **New / Edit / Delete entry** with rich-text (bold, italic, lists, headings, code blocks)
- [ ] **Tag entries** (create, rename, delete tags with colors)
- [ ] **Full-text search** across title and body
- [ ] **Calendar view** — browse entries by date
- [ ] **Mood picker** on each entry
- [ ] **Dark / Light / System theme**
- [ ] **Passphrase lock** — derives key with Argon2id; encrypts DB via SQLCipher
- [ ] **Auto-lock** after configurable idle timeout
- [ ] **Export** — single entry or all entries as Markdown or JSON
- [ ] **Responsive layout** — adapts between phone, tablet, and desktop widths

### v1.1

- [ ] **Image attachments** (stored as blobs or file references)
- [ ] **Streak / habit tracking** — consecutive days with entries
- [ ] **Reminder notifications** (Tauri notification plugin)
- [ ] **Backup & restore** — encrypted `.jrnl` archive

### v2.0 (cloud-optional)

- [ ] **End-to-end encrypted sync** via optional Go backend
- [ ] **Conflict-free merge** (CRDT or last-write-wins per field)
- [ ] **Multiple vaults** (separate DB files)

---

## 7. API / IPC Contract

All commands follow the pattern: `invoke<ReturnType>(command, args?)`.

### Entry commands

```typescript
// List entries (paginated, excludes soft-deleted)
invoke<Entry[]>('list_entries', {
  page: number,      // 1-based
  pageSize: number,  // default 20
  tagId?: string,
  query?: string,    // FTS query
  dateFrom?: number, // unix timestamp
  dateTo?: number,
})

// Get single entry with tags
invoke<EntryDetail>('get_entry', { id: string })

// Create entry — returns new entry id
invoke<string>('create_entry', {
  title: string,
  body: string,       // Tiptap JSON
  bodyText: string,   // plain text for FTS
  mood?: string,
  tagIds: string[],
})

// Update entry
invoke<void>('update_entry', {
  id: string,
  title?: string,
  body?: string,
  bodyText?: string,
  mood?: string,
  tagIds?: string[],
})

// Soft-delete entry
invoke<void>('delete_entry', { id: string })

// Search (FTS5)
invoke<Entry[]>('search_entries', { query: string })
```

### Tag commands

```typescript
invoke<Tag[]>('list_tags')
invoke<string>('create_tag', { name: string, color: string })
invoke<void>('update_tag', { id: string, name?: string, color?: string })
invoke<void>('delete_tag', { id: string })
```

### Settings / security commands

```typescript
invoke<void>('set_passphrase', { passphrase: string })      // first-time setup
invoke<void>('change_passphrase', { old: string, next: string })
invoke<void>('unlock', { passphrase: string })               // open encrypted DB
invoke<void>('lock')                                         // close & zero key
invoke<string>('get_setting', { key: string })
invoke<void>('set_setting', { key: string, value: string })
```

### TypeScript interfaces

```typescript
// src/types/index.ts

export interface Entry {
  id: string;
  title: string;
  body: string;
  bodyText: string;
  mood: string | null;
  weather: string | null;
  createdAt: number;  // unix timestamp (seconds)
  updatedAt: number;
  tags: Tag[];
}

export interface Tag {
  id: string;
  name: string;
  color: string;
}

export interface EntryDetail extends Entry {
  attachments: Attachment[];
}

export interface Attachment {
  id: string;
  entryId: string;
  filename: string;
  mimeType: string;
  size: number;
  createdAt: number;
}
```

---

## 8. UI/UX Guidelines

### Layout breakpoints

| Breakpoint | Width | Layout |
|------------|-------|--------|
| Phone | < 640 px | Single-column; bottom nav bar |
| Tablet | 640–1024 px | Two-column (sidebar + content) |
| Desktop | > 1024 px | Three-column (sidebar + list + editor) |

Use Tailwind responsive prefixes (`sm:`, `md:`, `lg:`) exclusively for breakpoints.

### Adaptive sidebar

```
Phone:    [  Editor / List  ]    ← no sidebar; hamburger opens drawer
Tablet:   [ Tags │  Entries / Editor  ]
Desktop:  [ Tags │  Entry List │  Editor  ]
```

### Accessibility

- All interactive elements must have `aria-label` or visible label.
- Keyboard navigation: `Tab`, `Shift+Tab`, `Enter`, `Space`, `Escape`.
- Color contrast ≥ 4.5:1 (WCAG AA).
- Touch targets ≥ 48×48 dp on Android.

### Theming

Store theme preference in `settings` table (`key = 'theme'`; values: `'light'`, `'dark'`, `'system'`).  
Apply via `document.documentElement.classList` toggling `dark` (Tailwind dark mode: `class`).

---

## 9. Security

| Concern | Mitigation |
|---------|------------|
| Data at rest | SQLCipher AES-256-CBC; key derived from passphrase with Argon2id (m=64MB, t=3, p=4) |
| Key in memory | Key wiped from Rust `Vec<u8>` on lock / app suspend |
| Auto-lock | Tauri global shortcut + idle timer; default 5 min |
| IPC surface | Only allow-listed Tauri commands are exposed (see `tauri.conf.json > allowlist`) |
| Dependency audit | `cargo audit` and `npm audit` run in CI on every PR |
| No telemetry | No network calls unless user explicitly enables cloud sync |
| Android storage | DB file stored in internal app storage (not external/SD card) |
| Export files | Exported Markdown/JSON placed in user-chosen directory; no auto-cloud-upload |

---

## 10. Testing Strategy

### Unit tests (Vitest)

- Test every custom hook in isolation using mock IPC responses.
- Test utility functions (date formatting, FTS query escaping, Tiptap JSON → plain-text).
- Coverage threshold: **80% lines** across `src/`.

```bash
npm run test:unit
```

### Integration tests (Vitest + msw)

- Mock Tauri `invoke` at the service layer.
- Test store actions end-to-end (create entry → state update → re-render).

```bash
npm run test:integration
```

### E2E tests (Playwright — desktop)

- Tests run against the full compiled Tauri binary on Linux (CI) and Windows (CI).
- Key scenarios:
  - Onboarding (set passphrase, create first entry)
  - Create / read / update / delete entry
  - Tag filtering
  - Full-text search returns correct entries
  - Theme switching persists across restarts
  - Export produces valid Markdown

```bash
npm run test:e2e
```

### Android tests (Detox — optional CI gate)

- Run on Android emulator (API 34).
- Smoke-test: launch → unlock → create entry → search.

```bash
npm run test:android
```

### Rust unit tests

```bash
cargo test --manifest-path src-tauri/Cargo.toml
```

---

## 11. CI/CD Pipeline

### `ci.yml` — runs on every PR to `main`

```yaml
jobs:
  lint-and-type-check:
    - npm ci
    - npm run lint
    - npm run type-check

  unit-tests:
    - npm run test:unit -- --coverage

  rust-checks:
    - cargo fmt --check
    - cargo clippy -- -D warnings
    - cargo audit
    - cargo test

  e2e-linux:
    runs-on: ubuntu-latest
    - npm run build
    - npm run test:e2e -- --project=linux

  e2e-windows:
    runs-on: windows-latest
    - npm run build
    - npm run test:e2e -- --project=windows
```

### `release.yml` — runs on `v*` tags

```yaml
jobs:
  build-linux:   ubuntu-latest  → .deb, .AppImage
  build-windows: windows-latest → .msi, .exe (NSIS)
  build-android: ubuntu-latest  → .apk, .aab
  publish:
    - Upload to GitHub Releases
    - Sign Android APK with keystore (secret)
    - Notarize Windows installer (optional)
```

---

## 12. Build & Release

### Development

```bash
# Install dependencies
npm install

# Start dev server with hot-reload (desktop)
npm run tauri dev

# Start dev server (Android — requires connected device or emulator)
npm run tauri android dev
```

### Production builds

```bash
# Desktop (runs on current OS)
npm run tauri build

# Android (requires Android SDK + NDK)
npm run tauri android build
# → outputs src-tauri/gen/android/app/build/outputs/apk/
```

### Environment variables

| Variable | Purpose |
|----------|---------|
| `TAURI_PRIVATE_KEY` | Updater signing key (CI secret) |
| `TAURI_KEY_PASSWORD` | Updater key password (CI secret) |
| `ANDROID_KEYSTORE_PATH` | Android release keystore (CI secret) |
| `ANDROID_KEY_ALIAS` | Keystore alias (CI secret) |
| `ANDROID_STORE_PASSWORD` | Keystore password (CI secret) |

### Version bumping

Follow [Semantic Versioning](https://semver.org/).  
Update version in **both** `package.json` and `src-tauri/Cargo.toml` before tagging.

```bash
# Example
git tag v1.0.0
git push origin v1.0.0   # triggers release.yml
```

---

## 13. Coding Conventions

### TypeScript / React

- **No `any`** — use `unknown` and narrow types.
- **Named exports** for components; default export only for page-level routes.
- **Hooks prefix** — all custom hooks must start with `use`.
- **Service layer** — all `invoke` calls live in `src/services/`; components never call `invoke` directly.
- **Error handling** — every `invoke` call is wrapped in try/catch; surface errors via a toast notification store, not `console.error`.
- **File naming** — `PascalCase` for components, `camelCase` for hooks/utils, `kebab-case` for CSS modules.
- Prefer `const` over `let`; never use `var`.

### Rust

- Follow `rustfmt` defaults (enforced in CI).
- Run `cargo clippy -- -D warnings` before committing.
- Every public function in `commands/` must have a doc comment.
- Return `Result<T, String>` from Tauri commands — the `String` is the user-facing error message.
- Use `thiserror` for internal error types.

### SQL / Migrations

- Migration filenames: `NNN_description.sql` (zero-padded, e.g. `003_attachments.sql`).
- Migrations run in ascending numerical order at startup.
- Never modify an already-merged migration — always add a new one.
- Always include `IF NOT EXISTS` / `IF EXISTS` guards.

### Git

- Branch naming: `feature/<slug>`, `fix/<slug>`, `chore/<slug>`.
- Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/): `feat:`, `fix:`, `docs:`, `test:`, `chore:`, `refactor:`.
- Every PR must have at least one passing CI run before merge.
- Squash-merge PRs to keep `main` history linear.

---

## 14. Contribution Workflow

1. Fork the repo (external contributors) or create a branch (team members).
2. Follow the [Coding Conventions](#13-coding-conventions).
3. Write or update tests for your changes.
4. Run `npm run lint && npm run type-check && npm run test:unit` locally — all must pass.
5. Open a PR targeting `main`; fill in the PR template.
6. CI must be green before review.
7. At least one code review approval is required before merging.
8. Maintainer squash-merges the PR.

---

*Last updated: 2026-03-20 — agent.md v1.0*
