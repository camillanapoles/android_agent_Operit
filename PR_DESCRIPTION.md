# Multilingual preferredLanguage for package descriptions

## Problem

Currently, `SystemPromptConfig.kt` uses a binary `useEnglish: Boolean` flag to decide
whether to show package/tool descriptions in English or Chinese:

```kotlin
val preferredLanguage = if (useEnglish) "en" else "zh"
```

When `useEnglish = false` (which happens for ALL non-English locales including pt-BR, ko, es, ms, id),
package descriptions resolve to Chinese — even though `LocalizedText.resolve()` has a perfectly
good multilingual fallback chain.

## Root Cause

`useEnglish` is computed as:
```kotlin
val useEnglish = LocaleUtils.getCurrentLanguage(context).lowercase().startsWith("en")
```

This is `false` for every locale except English, causing **all community packages** (which typically
have `{"zh": "...", "en": "..."}` METADATA) to display in Chinese.

## Fix

Replace the binary flag with the actual locale:

```kotlin
val preferredLanguage = LocaleUtils.getCurrentLanguage(context)
```

This makes `LocalizedText.resolve()` try: `pt-BR → pt → default → en → zh`
instead of always starting from `zh`.

## Changes

| File | Change |
|------|--------|
| `SystemPromptConfig.kt` | `if (useEnglish) "en" else "zh"` → `LocaleUtils.getCurrentLanguage(context)` |
| `ConversationService.kt` + 9 files | `useEnglish` → `preferredLanguage` pattern |
| `values-pt-rBR/strings.xml` | Filled ~129 missing keys from `values-en` base |
| ToolPkg `i18n/index.js` (5 pkgs) | Added `pt → en-US` in `normalizeLocale()` fallback |

## Impact

Benefits **ALL supported non-zh/en locales**:
- 🇧🇷 Portuguese (Brazil)
- 🇰🇷 Korean
- 🇪🇸 Spanish
- 🇲🇾 Malay
- 🇮🇩 Indonesian

## Testing

- [x] Gap analysis: 7179 default vs 7050 pt-rBR → 129 missing filled
- [x] Syntax validation: balanced braces, correct imports
- [ ] Full Gradle build (pending CI)
- [ ] Runtime test with device locale = pt-BR

## Backward Compatibility

- `LocalizedText.resolve()` fallback chain is unchanged (`locale → en → zh`)
- Chinese users still see Chinese (their locale resolves to `zh` correctly)
- English users still see English
- No breaking changes to METADATA format
