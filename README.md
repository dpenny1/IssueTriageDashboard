# CentralView Issue Triage Hub

A collection of single-page GitHub issue dashboards for tracking, triaging, and reporting on project issues. All dashboards run entirely in the browser — no server required.

## Live Dashboards

| Dashboard | File | Description |
|-----------|------|-------------|
| 🛡️ CIS | `cis.html` | CS Issues Hub — full triage with label-based tabs, ZenHub pipeline support, exception tracking, and weekly report |
| 🔍 Power Search | `powersearch.html` | General-purpose issue dashboard with search, filters, and label grouping |
| 🌐 Customer Portal | `customerportal.html` | Issues dashboard for the Customer Portal project |

Open `index.html` to navigate between dashboards.

## Features

All dashboards share a common set of core features:

- **GitHub API integration** — fetches open and recently closed issues directly from GitHub
- **Token Vault** — encrypted (AES-GCM, PBKDF2) token storage in `localStorage`; tokens saved in one dashboard are available across all others
- **Session token pass-through** — tokens unlocked on the home page vault carry into any dashboard automatically
- **Project profiles** — save named repo configurations (individual repos or multi-repo groups) so you don't re-enter them each visit
- **Stat cards** — at-a-glance counts for open, overdue, unassigned, and recently closed issues
- **Tabs** — All Open · New (<3d) · Overdue (>7d) · Unassigned · By Label · Recently Closed
- **Search & filters** — full-text search plus dropdowns for label, repo, and assignee
- **Sortable tables** — click any column header to sort
- **By Label view** — issues grouped under their label with color-coded headers pulled from GitHub label colors
- **Auto-refresh** — dashboard data refreshes automatically in the background every 20 minutes
- **Cache** — issues are cached in `localStorage` so the dashboard loads instantly on repeat visits while a background refresh runs

### CIS-specific features
- Label-based tab views (Exceptions, Production Patch, UAT Patch, CS Items, etc.)
- ZenHub pipeline integration
- Parent/sub-issue tracking
- Weekly report tab
- Release triage modal
- Mismatch scanning

## Setup

1. Open `index.html` in a browser (or serve the folder with any static file server)
2. Click a dashboard card
3. Enter your **GitHub personal access token** — needs `repo` scope (read-only is fine for viewing)
4. Add a project profile: enter your `owner/repo` (e.g. `myorg/powersearch`)
5. Click **Load Dashboard**

### Token Vault

The vault lets you save tokens under a friendly name, encrypted with a master password. To use it:

1. Click the **🔒 Vault** button next to the token field
2. Enter a master password to unlock (first use creates the vault)
3. Click **＋ Save** to store a token, or click an existing token name to fill the field

Tokens saved in the vault are available across all dashboards. The vault password is never stored — only the encrypted token data is saved in `localStorage`.

## Storage Keys

Each dashboard uses isolated `localStorage` keys to avoid conflicts:

| Dashboard | Cache key | Profiles key |
|-----------|-----------|--------------|
| CIS | `cis_issues_cache_v1` | `cis_profiles_v2` |
| Power Search | `ps_issues_cache_v1` | `ps_profiles_v1` |
| Customer Portal | `cp_issues_cache_v1` | `cp_profiles_v1` |
| Shared vault | `cis_vault_v1` | — |
