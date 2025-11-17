---
status: complete
created_at: 2025-11-17T14:51:28+00:00
requester: user
context_links: []
related_ticket: null
related_plan: null
related_operation: null
date: 2025-11-17T14:51:28+00:00
git_commit: cdcc21f371556c4f7bdb7a3148a48aa5f77a075d
branch: stable
repository: profilarr-database
topic: "Language preferences and profile applicability"
tags: [research, codebase, language-preferences, profiles]
last_updated: 2025-11-17
---

# Research: Language preferences and profile applicability

**Date**: 2025-11-17T14:51:28+00:00
**Researcher**: planning-specialist
**Git Commit**: cdcc21f371556c4f7bdb7a3148a48aa5f77a075d
**Branch**: stable
**Repository**: profilarr-database

## Research Question

"devi trovare se definiamo preferenxze per dei linguaggi, come le definiamo e a quali profili si applicano" — identify how language preferences are defined today and which profiles reference them. Context: upcoming plan to add preferences for releases that are in Italian and/or English.

## Summary

Language preferences in this repository are defined in two places:
1. **Profiles (`profiles/*.yml`)**: every profile ends with `language: must_original`, constraining upgrades to releases that preserve the original-language audio track. This is applied consistently across all 11 shipping profiles.
2. **Custom formats (`custom_formats/Not English.yml`, `Not Only English.yml`, `Not Only English (Missing).yml`)**: these custom formats use `type: language` conditions with `language: english` to control matching logic around English-language tracks, either requiring or excluding English and optionally detecting dual-audio releases.

No additional language settings exist under `media_management/`, `scripts/`, or `templates/`, and there are no thought documents covering language preferences.

## Detailed Findings

### Profile Language Constraint (`profiles/*.yml`)
- **Files**: all 11 profile definitions (e.g., `profiles/1080p Balanced.yml`, `profiles/2160p Quality.yml`, `profiles/720p Quality.yml`).
- **Structure**: each file concludes with
  ```yaml
  upgrade_until:
    ...
  language: must_original
  ```
  (see `profiles/1080p Balanced.yml:227-233`).
- **Behavior**: the `language` key is a peer to `upgrade_until`, indicating an upgrade filter rather than a scoring component. The value `must_original` enforces that acceptable upgrades must include the source-language audio track. This setting is repeated verbatim across all profiles, meaning every quality stack currently inherits the same original-language requirement.

### Custom Format Language Rules
1. **`custom_formats/Not English.yml`** (`lines 1-19`)
   - Uses two `type: language` conditions:
     - `exceptLanguage: true`, `language: english`, `required: true` → matches releases lacking English audio.
     - `exceptLanguage: false`, `language: english`, `negate: true`, `required: true` → explicitly fails if English is present (reinforces the exclusion).
   - Purpose: identify non-English releases while allowing dual-audio as long as English is absent.

2. **`custom_formats/Not Only English.yml`** (`lines 1-19`)
   - Similar structure but second condition lacks `negate: true`, so it requires an English track alongside the non-English requirement—meaning dual audio where English is included but not the sole language.
   - Prevents single-language English releases while allowing combinations (e.g., English + another language).

3. **`custom_formats/Not Only English (Missing).yml`** (`lines 1-19`)
   - Only one `type: language` condition (`language: english`, `required: true`) ensuring English is present.
   - Adds a `type: release_title` condition (`pattern: Dual Audio`) to infer multi-language availability when metadata fails to mark it explicitly.

These custom formats can be referenced within profile `custom_formats` blocks to adjust scoring for desired language scenarios. The existing profiles list numerous format names; none currently point to the “Not English/Not Only English” formats, so adoption would require adding the relevant entries to specific profile scoring lists.

## Code References
- `profiles/1080p Balanced.yml:19-233` – Profile upgrade parameters culminating in `language: must_original`.
- `profiles/1080p Compact.yml:19-308` – Same language requirement within compact profile.
- `profiles/2160p Quality.yml:19-321` – UHD profile with identical language constraint.
- `profiles/720p Quality.yml:19-212` – Lowest resolution profile sharing the same rule.
- `custom_formats/Not English.yml:1-19` – Exclusion of English audio tracks.
- `custom_formats/Not Only English.yml:1-19` – Dual-language enforcement with English included.
- `custom_formats/Not Only English (Missing).yml:1-19` – Detection fallback for dual audio plus English.

## Architecture Documentation
- **Language Fields**: `language` key exists only in profile root-level configuration (after `upgrade_until`) and in custom format conditions via `type: language` entries.
- **Value Semantics**: profile-level `language` accepts `must_original`; custom formats specify explicit language names (`english`) combined with `exceptLanguage`, `negate`, and `required` flags to shape match logic.
- **Interaction Model**: profiles rely on custom formats for scoring and `upgrade_until` for quality ceilings, with `language` acting as a final eligibility filter. Custom formats provide reusable definitions for language-specific scenarios that profiles can include by name to influence upgrade scoring.

## Historical Context (from thoughts/)
- No thought documents exist regarding language preferences; repository lacks a `thoughts/` tree.

## Related Research
- None available; this is the first research entry on language preferences.

## Open Questions
- Which profiles (if any) should incorporate the existing “Not English / Not Only English” custom formats to influence scoring? (Current research only documents status quo.)
- How will forthcoming Italian/English preferences be structured within the existing `language`/custom-format framework? (Requires future planning once requirements are set.)
