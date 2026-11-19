---
name: r8-release
description: Ship Android release builds that R8 doesn't break — keep rules that libraries ship as consumer-rules.pro, the kotlinx-serialization block, keepnames for nav keys used as screen names, removing blanket keeps one at a time, mapping upload, and testing on the minified variant. Use when touching ProGuard/R8 rules, adding a @Serializable type, a library module, or before a release.
---

# R8 release builds

## Rules belong with the code that needs them

A library module that needs a rule ships it in `consumer-rules.pro` (`consumerProguardFiles`). The app's `proguard-rules.pro` stays small.

## Must-haves

- The official kotlinx-serialization block, plus `-keepattributes Signature, *Annotation*, InnerClasses, EnclosingMethod`. A missing keep for a new `@Serializable` DTO or nav key is a **release-only** `SerializationException`.
- `-keepnames class * implements androidx.navigation3.runtime.NavKey` when screen names come from `this::class.simpleName`. Without it, screens arrive in analytics and monitoring as `j83`.
- Anything reached by reflection or JNI.

## Never

- `-keep class ** { *; }` or blanket `-keep class io.ktor.** { *; }`-style rules: they disable shrinking for the whole library.
- Removing a blanket keep without a test. Remove them **one at a time, safest first**, and after each run a real round trip (sign in, load a list, save) on the minified `prodRelease`.

## Mappings

Upload `mapping.txt` for every release from the module that runs R8 (the Android app), tagged with the build id the crash and monitoring tools report. Without it, stack traces are obfuscated.

## Test on the real variant

`./gradlew :apps:app:androidApp:assembleProdRelease`, install, run the E2E flows. Debug builds skip R8 and hide these bugs.

The `r8-analyzer` Android skill, if installed, finds redundant or over-broad rules.
