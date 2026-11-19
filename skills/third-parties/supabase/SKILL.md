---
name: supabase
description: Use Supabase from a KMP app with supabase-kt — a wrapper library over the Auth plugin, an encrypted session manager per platform, a lazily built client off the main thread, keys resolved from local.properties per environment, edge functions called through one API client with token refresh, and RLS as the real security boundary. Use when adding Supabase auth, an edge function call, or Supabase config.
---

# Supabase (supabase-kt)

## Wrap it once

`libraries/network/supabase-auth` wraps supabase-kt's `Auth` plugin behind a `SupabaseAuth` interface. Services depend on the interface; nothing else imports supabase-kt. supabase-kt already does session persistence and token refresh on every target: don't hand-roll them.

## Startup cost

`createSupabaseClient` can cost 0.5 s+. Build it `by lazy(LazyThreadSafetyMode.SYNCHRONIZED)`, observe it with `flowOn(dispatcher)`, and await the initial session in a startup initializer (see `feature-startup`), so Main never builds it.

## Session storage

```kotlin
// each platform source set annotates its own @Single SessionManager; the client takes SessionManager?
// android: SettingsSessionManager(SharedPreferencesSettings(EncryptedSharedPreferences(... AES256_GCM)))
// ios:     SettingsSessionManager(KeychainSettings(service = "<app>.auth"))
// jvm:     none → supabase-kt's default (unencrypted); say so
```

The Android `Context` comes from an `androidx.startup` initializer declared in the library's manifest.

## Config

- `local.properties`: `supabase.stgProjectRef` (→ `https://<ref>.supabase.co`) or `supabase.stgUrl` (wins; e.g. local `http://127.0.0.1:54321`), plus `supabase.stgAnonKey`. Same with `prod`.
- Strip whitespace at build time (a newline in the key breaks the `Authorization` header). `require(url.isNotBlank())` instead of falling back to localhost.
- The anon key is public by design; **row-level security is the boundary**. The service-role key never ships in an app.

## Edge functions

One `EdgeApiClient` (`@Single`, built by the app) with reified `get/post/put/delete`:

- `Authorization: Bearer <anon key>` + the user's access token; on `401`, refresh once and retry.
- Route prefix `${supabaseUrl}/functions/v1/<function>/`; routes as constants in each remote data source.
- Its exceptions embed the raw response body: turn them into `OfflineException` or `ReasonException` in the client (see `feature-network`), never show the message.

## Sign-in with an ID token

`auth.signInWith(IDToken) { idToken = token; provider = Google }` (see `google-sign-in`). Enable the provider in the dashboard with the same web client id.
