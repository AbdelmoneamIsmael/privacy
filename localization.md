# Localization in Digital Atlas

**Document purpose:** Explain how localization works in the Digital Atlas Flutter app, why `easy_localization` was chosen, how to use translations in code, and how to generate type-safe keys.

**Last updated:** June 2026

---

## Table of Contents

1. [Overview](#1-overview)
2. [Why easy_localization](#2-why-easy_localization)
3. [Project Structure](#3-project-structure)
4. [App Setup](#4-app-setup)
5. [How to Use Translations](#5-how-to-use-translations)
6. [Switching Language](#6-switching-language)
7. [Adding New Strings](#7-adding-new-strings)
8. [Generating locale_keys.g.dart](#8-generating-locale_keysgdart)
9. [Key Source Files](#9-key-source-files)
10. [Summary](#10-summary)

---

## 1. Overview

Digital Atlas supports **English (`en`)** and **Arabic (`ar`)**. All user-facing text is stored in JSON translation files and loaded at runtime by [`easy_localization`](https://pub.dev/packages/easy_localization).

The flow is:

```
assets/translations/en.json  ─┐
assets/translations/ar.json  ─┼─► EasyLocalization ─► .tr() ─► UI text
lib/translations/locale_keys.g.dart (type-safe keys)
```

- **Translation files** hold the actual strings per language.
- **`locale_keys.g.dart`** holds generated constants so you avoid typos in key names.
- **`.tr()`** resolves a key to the string for the current locale.

---

## 2. Why easy_localization

We chose `easy_localization` over alternatives (e.g. Flutter’s built-in ARB/`gen-l10n`, `flutter_i18n`, `slang`) for these reasons:

| Reason | Benefit |
|--------|---------|
| **JSON translation files** | Easy to read, edit, and share with translators. No code generation step required just to add a string. |
| **Runtime locale switching** | Users can switch between English and Arabic without restarting the app. |
| **Simple API** | Call `.tr()` on any key string — minimal boilerplate in widgets. |
| **Type-safe keys (optional)** | The `generate` command produces `LocaleKeys` constants for autocomplete and compile-time safety. |
| **RTL-friendly** | Works well with Arabic; Flutter handles layout direction when locale changes. |
| **Fits our stack** | Integrates cleanly with `MaterialApp.router`, BLoC, and persisted language preference via `AppEnv` / Hive. |

For a bilingual app with JSON-based content and in-app language toggle, `easy_localization` keeps setup small and the developer experience straightforward.

---

## 3. Project Structure

```
assets/translations/
├── en.json          # English strings
└── ar.json          # Arabic strings

lib/translations/
└── locale_keys.g.dart   # Auto-generated key constants (do not edit manually)
```

### Translation file format

Keys use **dot notation** for nested JSON objects. Example from `en.json`:

```json
{
  "app": {
    "title": "Digital Atlas"
  },
  "nav": {
    "home": "Home",
    "plants": "Plants"
  },
  "see_all": "See All"
}
```

- Nested key `nav.home` maps to `"Home"`.
- Top-level key `see_all` maps to `"See All"`.

The Arabic file (`ar.json`) must use the **same key structure** with translated values.

---

## 4. App Setup

### Dependencies (`pubspec.yaml`)

```yaml
dependencies:
  flutter_localizations:
    sdk: flutter
  intl: any
  easy_localization: ^3.0.8

flutter:
  assets:
    - assets/translations/
```

### Initialization (`lib/main.dart`)

1. Call `EasyLocalization.ensureInitialized()` before `runApp`.
2. Wrap the app with `EasyLocalization`, pointing to the translations folder.

```dart
await EasyLocalization.ensureInitialized();

runApp(
  EasyLocalization(
    supportedLocales: const [Locale('en'), Locale('ar')],
    path: 'assets/translations',
    fallbackLocale: Locale(appEnv.language.name),
    startLocale: Locale(appEnv.language.name),
    child: const DigitalAtlasApp(),
  ),
);
```

### MaterialApp wiring (`lib/digital_atlas_app.dart`)

`MaterialApp.router` must receive locale delegates from `EasyLocalization`:

```dart
MaterialApp.router(
  key: ValueKey(locale.languageCode), // rebuild when locale changes
  locale: locale,
  supportedLocales: context.supportedLocales,
  localizationsDelegates: context.localizationDelegates,
  // ...
)
```

The `ValueKey` on `MaterialApp` forces a rebuild when the user switches language so the whole UI updates.

---

## 5. How to Use Translations

### Step 1 — Import the generated keys

```dart
import 'package:digital_atlas/translations/locale_keys.g.dart';
import 'package:easy_localization/easy_localization.dart';
```

### Step 2 — Use a key and call `.tr()`

**Pattern A — Pass key to a shared widget (preferred for buttons/headers)**

Shared widgets like `AppButton` and `AppSectionHeader` accept a `String title` and call `.tr()` internally:

```dart
AppButton(
  title: LocaleKeys.drawer_logout,
  onPressed: () { /* ... */ },
)

AppSectionHeader(
  title: LocaleKeys.home_short_videos_title,
  actionText: LocaleKeys.see_all,
)
```

**Pattern B — Call `.tr()` at the call site**

Use this when the widget does not translate for you (e.g. labels, tooltips):

```dart
BottomNavItem(
  tooltip: LocaleKeys.nav_home.tr(),
  label: LocaleKeys.nav_home.tr(),
)
```

### Rules of thumb

- Always use `LocaleKeys.*` constants — never hardcode raw strings like `'nav.home'`.
- Any string shown to the user should go through `.tr()` (directly or via a shared widget).
- `LocaleKeys` values are **keys**, not translated text. `LocaleKeys.nav_home` is `'nav.home'`; `.tr()` returns `"Home"` or `"الرئيسية"` depending on locale.

---

## 6. Switching Language

Language switching is handled by `AppCubitCubit.setLanguage()`:

1. Toggle between `ApplicationLanguage.en` and `ApplicationLanguage.ar`.
2. Persist the choice in `AppEnv` / Hive.
3. Call `context.setLocale(Locale(language.name))` from `easy_localization`.

The drawer exposes this via the **Switch Language** button (`LocaleKeys.drawer_switch_language`).

When locale changes:

- `EasyLocalization` loads the matching JSON file.
- `MaterialApp` rebuilds (via `ValueKey`).
- All widgets using `.tr()` show the new language.

---

## 7. Adding New Strings

1. **Add the key and English text** in `assets/translations/en.json`.
2. **Add the same key** with Arabic text in `assets/translations/ar.json`.
3. **Regenerate** `locale_keys.g.dart` (see next section).
4. **Use the new key** in code: `LocaleKeys.your_new_key.tr()` or pass it to a widget that calls `.tr()`.

Example — adding a welcome message:

**en.json**
```json
{
  "home": {
    "welcome": "Welcome back"
  }
}
```

**ar.json**
```json
{
  "home": {
    "welcome": "مرحباً بعودتك"
  }
}
```

After generation, use:

```dart
Text(LocaleKeys.home_welcome.tr())
```

---

## 8. Generating locale_keys.g.dart

After editing JSON files, regenerate the type-safe keys file. Run this from the **project root** (`digital_atlas/`):

```bash
dart run easy_localization:generate -S "assets/translations" -O "lib/translations" -o "locale_keys.g.dart" -f keys
```

### Command flags

| Flag | Value | Meaning |
|------|-------|---------|
| `-S` | `assets/translations` | **S**ource folder with JSON translation files |
| `-O` | `lib/translations` | **O**utput folder for generated Dart code |
| `-o` | `locale_keys.g.dart` | **O**utput file name |
| `-f` | `keys` | **F**ormat: generate a `LocaleKeys` class with `static const` strings |

### When to run it

- After adding, renaming, or removing translation keys in JSON.
- Before committing if JSON and `locale_keys.g.dart` would otherwise be out of sync.

> **Do not edit `locale_keys.g.dart` by hand.** It is overwritten on each generate run.

---

## 9. Key Source Files

| File | Role |
|------|------|
| `assets/translations/en.json` | English strings |
| `assets/translations/ar.json` | Arabic strings |
| `lib/translations/locale_keys.g.dart` | Generated key constants |
| `lib/main.dart` | `EasyLocalization` init and wrapper |
| `lib/digital_atlas_app.dart` | `MaterialApp` locale delegates |
| `lib/core/bloc/app_cubit/cubit/app_cubit_cubit.dart` | Language toggle + persistence |
| `lib/core/widgets/app_button.dart` | Example: `.tr()` inside shared widget |
| `lib/core/widgets/app_section_header.dart` | Example: `.tr()` inside shared widget |

---

## 10. Summary

- Translations live in **`assets/translations/*.json`** (one file per locale).
- Use **`LocaleKeys`** + **`.tr()`** in widgets for type-safe, localized UI text.
- **`easy_localization`** was chosen for JSON files, runtime switching, simple API, and good fit with Arabic/RTL.
- After changing JSON keys, run:

  ```bash
  dart run easy_localization:generate -S "assets/translations" -O "lib/translations" -o "locale_keys.g.dart" -f keys
  ```
