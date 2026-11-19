---
name: firebase
description: Use Firebase from a KMP app that also targets web and desktop — Auth and Storage through their REST APIs with Ktor instead of the platform SDKs, resumable chunked uploads streamed from disk, a single-flight token refresh, each Firebase product in its own Gradle module, and App Distribution for tester builds. Use when adding Firebase Auth, Storage, Analytics or App Distribution.
---

# Firebase

## REST over SDKs when you target wasm or desktop

The Android/iOS SDKs don't exist for wasm and JVM. Firebase Auth (`identitytoolkit.googleapis.com`, `securetoken.googleapis.com`) and Storage (`firebasestorage.googleapis.com`) have REST APIs that work everywhere through Ktor.

- One Gradle module per product (`firebase-auth`, `firebase-storage`): one product that can't compile for a target doesn't break the others.
- They receive `Lazy<HttpClient>` and `Lazy<Json>` from the app's graph.

## Auth

- Credentials in the **request body**, never `parameter(...)` (query strings land in logs).
- Store the refresh token in **encrypted** storage (Keystore-backed on Android, Keychain on iOS), never plain settings.
- `getOrRenewIdToken()` behind a `Mutex`: concurrent calls share one refresh. Every client (REST, gRPC, functions) uses it, never a raw cached token.
- Sign in with a provider token: `accounts:signInWithIdp` (see `google-sign-in`).
- A stored token for a deleted user: load the user with a timeout and sign out if it never arrives.

## Storage: resumable upload

- Start a session (`X-Goog-Upload-Protocol: resumable`), then send chunks with `X-Goog-Upload-Command: upload` (last: `upload, finalize`) and `X-Goog-Upload-Offset`.
- Non-final chunks are a multiple of 256 KiB (e.g. 20 MB).
- Stream from disk with `expect fun PlatformFile.buffered(): Source`; never read the whole file into memory.
- On failure, query the session for the committed offset and resume from there.

## Analytics

Firebase/Google Analytics is one consumer in the analytics fan-out (see `feature-analytics`); screen views map to `screen_view { screen_name }`.

## App Distribution

- CI uploads tester builds (see `codemagic`); testers in a group get them in the App Tester app.
- A printed release link may mask the app id; get invite links from the console and open them before sharing. Invite links expire.

## Config

`google-services.json` / `GoogleService-Info.plist` are gitignored and written by CI from secrets; one Firebase project per environment.
