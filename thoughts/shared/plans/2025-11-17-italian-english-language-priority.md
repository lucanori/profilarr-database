---
status: draft
created_at: 2025-11-17T15:45:00Z
requester: user
context_links:
  - thoughts/shared/research/2025-11-17-language-preferences.md
related_ticket: null
related_research: thoughts/shared/research/2025-11-17-language-preferences.md
related_operation: null
---

# Italian/English Language Preference Implementation Plan

## Overview
Prioritize releases with dual Italian+English audio tracks, fall back to Italian-only, and use English-only as a last resort by leveraging custom format scoring across every Radarr/Sonarr profile while dropping the restrictive `language: must_original` filter.

## Current State Analysis
- Three custom formats (`custom_formats/Not English*.yml`) gate on English audio only and are unused by any profile scoring block.
- All 11 profiles (`profiles/*.yml`) end with `language: must_original`, preventing upgrades that drop the source-language audio even when Italian-only releases are desired (e.g., `profiles/1080p Balanced.yml:233`).
- No profile scoring tiers reference language-specific formats, so language never influences selection beyond the global original-language constraint.
- Supporting regex (`regex_patterns/Dual Audio.yml`) already detects generic "Dual Audio"/"MULTi" markers and can be reused for Italian/English multi-track inference.

## Desired End State
- Custom formats exist for each relevant combination:
  1. Italian+English dual audio (hard prefer)
  2. Italian-only (secondary)
  3. English-only (tertiary)
  4. Penalty format for releases missing both Italian and English (AND condition)
- Updated versions of current `Not English*` formats incorporate Italian logic as requested.
- Every profile applies the new scoring ladder (Radarr and Sonarr sections as applicable) so upgrades adhere to the ITA+ENG → ITA → ENG ordering.
- The `language: must_original` key is removed from all profiles, allowing scoring to drive selection.

### Key Discoveries:
- `custom_formats/Not English.yml:1-19` and siblings only reason about English audio.
- `profiles/*` files (e.g., `profiles/2160p Quality.yml:1-321`) share identical language filters and lack language scoring.
- `regex_patterns/Dual Audio.yml:1-117` already matches "Dual Audio"/"MULTi" tokens, useful for fallback detection when metadata lacks explicit Italian/English markers.

## What We're NOT Doing
- No changes to the quality tier definitions, score magnitudes unrelated to language, or provider-specific scores.
- No adjustments to scripts or templates outside the YAML data files.
- No automation of download client behavior beyond what profiles/custom formats already control.

## Implementation Approach
Create/extend language-aware custom formats first, then weave them into every profile's scoring (both Radarr and Sonarr blocks) with consistent score deltas, and finally remove the `must_original` guard so that the new scores dictate priority.

## Phase 1: Define & Update Language Custom Formats

### Overview
Extend existing `Not English*` definitions and add the Italian/English combinations with clear naming, condition logic, and regression tests.

### Changes Required:

#### 1. Update existing formats
**Files**:
- `custom_formats/Not English.yml`
- `custom_formats/Not Only English.yml`
- `custom_formats/Not Only English (Missing).yml`

**Changes**:
- Rename/descriptions to reflect Italian/English scope (e.g., "Not Italian or English").
- Adjust `type: language` conditions so that:
  - The formats detect absence of **both** Italian and English tracks (AND clause by combining `language: italian` + `language: english` with `negate: true`).
  - Dual-audio inference uses `Dual Audio` regex plus explicit Italian/English detection where metadata is incomplete.
- Add regression tests under the `tests` key with representative release titles (ITA+ENG, ITA-only, ENG-only, neither) to lock behavior.

#### 2. Add new custom formats
**Files**:
- `custom_formats/Italian and English.yml` (new)
- `custom_formats/Italian Only.yml` (new)
- `custom_formats/English Only.yml` (new)
- `custom_formats/Not Italian or English.yml` (if existing files are repurposed, ensure naming is unique; otherwise introduce separate penalty format)

**Changes**:
- `Italian and English`: require both languages via two `type: language` entries (`required: true`, `negate: false`) plus optional `release_title` regex referencing `Dual Audio` pattern for metadata gaps.
- `Italian Only`: require Italian (`language: italian`) and exclude English via `exceptLanguage: false`, `language: english`, `negate: true`.
- `English Only`: require English and optionally ensure Italian absent (unless coexisting Italian should elevate to the dual-audio format exclusively).
- `Not Italian or English`: penalize releases lacking **either** language using two `type: language` conditions with `negate: true`, `required: true`, ensuring the AND behavior the user requested (release hits the penalty only if both checks fail).
- Provide `tests` arrays describing sample release names/metadatas to validate each combination.

### Success Criteria:

#### Automated Verification:
- [ ] YAML schema validation passes (e.g., `python scripts/tierCreator.py --validate custom_formats`).
- [ ] New/updated custom formats include passing `tests` entries (run via Profilarr tooling, e.g., `profilarr --test custom_formats/Italian and English.yml`).

#### Manual Verification:
- [ ] Spot-check Sonarr/Radarr UI import of the new custom formats to ensure condition rendering is correct.
- [ ] Confirm naming/description clarity aligns with repository conventions.

---

## Phase 2: Apply Language Scores Across Profiles

### Overview
Inject the new custom formats into every profile (Radarr and Sonarr sections) with consistent scoring so the system prefers ITA+ENG over ITA, then ENG, and heavily penalizes any release missing both languages.

### Changes Required:

#### 1. Profile `custom_formats` blocks
**Files**: all `profiles/*.yml` (11 files total).

**Changes**:
- Add entries near the top of each `custom_formats` list with descending scores, e.g.:
  - `Italian and English` → +25,000 (highest after tier selectors)
  - `Italian Only` → +15,000
  - `English Only` → +5,000
  - `Not Italian or English` → -999,999 (or another strong penalty consistent with other "banned" formats)
- Ensure score magnitudes respect existing `minCustomFormatScore` thresholds so the additions influence upgrades without triggering unintended caps.

#### 2. `custom_formats_radarr` / `custom_formats_sonarr`
**Files**: same profile set (only those that currently define these sections).

**Changes**:
- Mirror the same scoring ladder inside Radarr/Sonarr subsections (where present) so movie- and series-specific evaluations behave identically.
- If a profile lacks a given section, add it only if language-specific scoring is necessary for that client; otherwise keep definitions in the global `custom_formats` block and document that Radarr/Sonarr pick them up.

#### 3. Documentation in profiles
**Files**: profiles updated above.

**Changes**:
- Update descriptions (if needed) to mention language prioritization so maintainers understand the new behavior.

### Success Criteria:

#### Automated Verification:
- [ ] YAML validation for profiles (same command as Phase 1, run against `profiles/`).
- [ ] Spot-check scoring order using Profilarr tooling (e.g., `profilarr profile score --profile "1080p Balanced" --mock-release Italian+English`).

#### Manual Verification:
- [ ] Inspect each profile to confirm the new language entries appear in the intended order (high → low).
- [ ] Verify sample releases in Radarr/Sonarr show the expected score differentials (ITA+ENG outranking ITA-only, etc.).

---

## Phase 3: Remove `language: must_original` and Final Validation

### Overview
Eliminate the original-language guard from every profile so the new scoring can elevate Italian-only releases even when the original work is English.

### Changes Required:

#### 1. Remove key
**Files**: all `profiles/*.yml`.

**Changes**:
- Delete the `language: must_original` block at the end of each file.
- Ensure no trailing whitespace/formatting issues remain.

#### 2. Final validation pass
**Files**: n/a (repository-level operation).

**Changes**:
- Run full schema/tests as in prior phases.
- Document the rationale in an operation record once implementation concludes (linking this plan and the research doc).

### Success Criteria:

#### Automated Verification:
- [ ] Repository-wide validation/lint (e.g., `python scripts/tierCreator.py --validate profiles custom_formats`).
- [ ] CI or pre-commit (if configured) runs clean.

#### Manual Verification:
- [ ] Confirm via Radarr/Sonarr UI that profiles no longer show the `must_original` filter.
- [ ] Validate upgrade previews now list Italian-only releases above English-only when dual audio is unavailable.

---

## Testing Strategy

### Unit Tests:
- Rely on the `tests` sections within each custom format to verify language condition logic (Italian+English, Italian-only, English-only, no Italian/English).

### Integration Tests:
- Use Profilarr tooling or Radarr/Sonarr test instances to score mock releases:
  1. Release with ITA+ENG audio should surpass existing top-scoring formats when quality is equal.
  2. Release with only Italian audio must outrank English-only equivalents.
  3. Release lacking both Italian and English receives the penalty and falls below `minCustomFormatScore` thresholds.

### Manual Testing Steps:
1. Import each updated profile into a Radarr/Sonarr test environment.
2. Evaluate sample releases (dual audio, ITA-only, ENG-only, other language) to confirm scoring order.
3. Attempt an upgrade on a title whose original language is English but has Italian-only releases; verify the system now accepts the Italian release while still penalizing releases without either language.

## Performance Considerations
- Minimal: updates are static YAML data. Ensure scoring additions do not significantly increase evaluation time (Profilarr handles dozens of custom formats already).

## Migration Notes
- Removing `language: must_original` may allow upgrades previously blocked; communicate this change to operators so they understand the new behavior.
- Encourage re-importing profiles to Radarr/Sonarr after changes so clients pick up the new scoring and custom formats.

## References
- Research source: `thoughts/shared/research/2025-11-17-language-preferences.md`
- Existing language formats: `custom_formats/Not English.yml`, `custom_formats/Not Only English.yml`, `custom_formats/Not Only English (Missing).yml`
- Profiles requiring updates: all files under `profiles/`
- Regex support: `regex_patterns/Dual Audio.yml`
