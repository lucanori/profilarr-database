---
status: complete
created_at: 2025-11-17T15:45:00Z
requester: user
context_links:
  - thoughts/shared/plans/2025-11-17-italian-english-language-priority.md
  - thoughts/shared/research/2025-11-17-language-preferences.md
related_ticket: null
related_research: thoughts/shared/research/2025-11-17-language-preferences.md
related_plan: thoughts/shared/plans/2025-11-17-italian-english-language-priority.md
---

# Italian/English Language Preference Implementation Operation

## Overview
Implemented language prioritization for Italian and English audio tracks across all profiles, replacing the restrictive `language: must_original` filter with custom format scoring that prefers dual Italian+English, then Italian-only, then English-only, while penalizing releases lacking both languages.

## Changes Implemented

### Phase 1: Define & Update Language Custom Formats
- Updated existing `Not English*` formats to detect absence of both Italian and English tracks
- Created new formats: `Italian and English.yml`, `Italian Only.yml`, `English Only.yml`, `Not Italian or English.yml`
- Added regression tests for each format

### Phase 2: Apply Language Scores Across Profiles
- Added language scoring to all 11 profiles with consistent deltas (+25k for dual, +15k for Italian-only, +5k for English-only, -999k penalty)
- Updated both Radarr and Sonarr sections where applicable

### Phase 3: Remove `language: must_original`
- Removed the original-language constraint from all profiles to allow scoring-driven selection

## Rationale
Based on research showing current profiles blocked Italian-only upgrades even when preferred, and existing custom formats only handled English logic. This implementation enables flexible language preferences while maintaining quality standards.

## References
- Plan: `thoughts/shared/plans/2025-11-17-italian-english-language-priority.md`
- Research: `thoughts/shared/research/2025-11-17-language-preferences.md`