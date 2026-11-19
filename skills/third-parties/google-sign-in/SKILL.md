---
name: google-sign-in
description: Google sign-in for a KMP app — Android Credential Manager with the web client id and a nonce, the ID token verified on the server (JWKS signature, iss, aud, exp, nonce, domain rule), the Android OAuth client set up for every signing key, a brokered web flow for iOS and desktop (no fallback on Android), and a sealed result where cancel shows nothing. Use when adding, debugging or reviewing Google sign-in on the client or the server.
---

# Google sign-in

## Shape

```kotlin
sealed interface GoogleSignInResult {
    data class Success(val user: User) : GoogleSignInResult
    data object Cancelled : GoogleSignInResult
    data class Error(val message: StringResource) : GoogleSignInResult
    data object NotAvailable : GoogleSignInResult
}
interface GoogleSignIn { fun isAvailable(): Boolean; suspend fun signIn(): GoogleSignInResult }
// each platform source set: @Single(binds = [GoogleSignIn::class]) class DefaultGoogleSignIn(…); web without a server flow binds none
```

## Android: Credential Manager with a nonce

```kotlin
val verifier = randomUrlSafe(32)
val option = GetGoogleIdOption.Builder()                    // or GetSignInWithGoogleOption.Builder(webClientId)
    .setServerClientId(webClientId).setFilterByAuthorizedAccounts(false)
    .setNonce(sha256Base64Url(verifier)).build()
val result = credentialManager.getCredential(activity, GetCredentialRequest(listOf(option)))
val idToken = GoogleIdTokenCredential.createFrom(result.credential.data).idToken
authRepository.signInWithGoogle(idToken, verifier)         // the server checks nonce == S256(verifier)
```

- `serverClientId` is the **web** client id. The token's `aud` is that id; `azp` is the Android client.
- The nonce ties the token to the app instance that asked for it: a token lifted from another app or log can't be replayed.
- Credential Manager needs an Activity context from the composition, not the application.
- `GetCredentialCancellationException` → `Cancelled`: no error.
- `NoCredentialException`, the developer-console error, or no Play Services → `Error`. No browser fallback on Android: the sheet already offers to add an account, and a phone without Play Services uses email. It saves a custom scheme, `singleTask`, `androidx.browser` and four activity hooks.

## The server verifies every token

1. **Signature** against Google's JWKS (`https://www.googleapis.com/oauth2/v3/certs`, RS256), keys cached by `kid`, refetched once when a `kid` is unknown.
2. `iss` is `accounts.google.com` or `https://accounts.google.com`; `aud` is the web client id; `exp` in the future; `email_verified` true.
3. `nonce` equals `S256(verifier)`.
4. The **domain rule** (hosted domain `hd` or email domain) **before** any account is created.

Skipping the signature is only safe for a token the server got itself from Google's token endpoint (code flow). It is never safe for a token a client sends. Test with a locally generated RSA key behind a swappable key source.

## Android OAuth client setup

- An Android OAuth client (package + SHA-1) **for each signing key**: debug, upload, and Play app signing, in the **same GCP project** as the web client and its consent screen.
- Missing or in another project, Credential Manager fails with a developer-console error, not a clear one.
- While the consent screen is in **Testing**, only its listed test users can sign in.

## Where the client id comes from

- The build, per environment: web client id in `local.properties` → BuildKonfig (or `buildConfigField`) → a `GoogleSignInKeyProvider` bound by the app. It's public, so it may also be committed.
- Don't fetch it from the server: it's public and fixed per environment, so a network call adds nothing. Writing it once per platform is fine.
- It must equal the server's configured web client id, or the server refuses the token (`aud`).
- `isAvailable() = webClientId.isNotBlank()`; the button is disabled until there is one. Callers take `GoogleSignIn?` and hide the button on a platform that binds none.

## iOS and desktop: a brokered web flow

The server runs the OAuth web flow and hands the app a **one-time code bound by PKCE**:

| Platform | Opens | Returns through |
|---|---|---|
| iOS | `ASWebAuthenticationSession` | callback scheme |
| Desktop | system browser | loopback listener on `127.0.0.1` |

So iOS and desktop don't need to return `NotAvailable`; only web without a server flow does.

## Checklist

- [ ] Server checks signature, iss, aud, exp, email_verified, nonce, domain rule
- [ ] Android clients for debug, upload and Play signing keys, same project as the web client
- [ ] Cancel shows nothing; developer-console and no-credential errors show an error
- [ ] Tests: server verifier with a local RSA key (bad signature, wrong aud, expired, nonce mismatch, wrong domain)
