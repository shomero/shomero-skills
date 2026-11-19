---
name: revenuecat
description: Integrate RevenueCat purchases-kmp for subscriptions in a KMP app — a mobileMain source set for the SDK with no-op desktop/web data sources, one use case per store operation, identity synced on login and logout, typed cancel handling, and entitlement taken from the server. Use when adding purchases, a paywall, restore, upgrade/downgrade, or entitlement checks.
---

# RevenueCat (purchases-kmp)

## Source sets

```kotlin
val mobileMain by creating { dependsOn(commonMain.get()) }
androidMain.get().dependsOn(mobileMain); iosMain.get().dependsOn(mobileMain)
applyDefaultHierarchyTemplate()          // or iosArm64Main → iosMain is cut
```

`mobileMain` holds a `PaymentRemoteMobileDelegate` with all SDK calls. Each platform source set declares its own `@Single(binds = [PaymentRemoteDataSource::class]) class DefaultPaymentRemoteDataSource`: Android and iOS delegate, desktop and web are no-op (`isAvailable = false`). Simpler than `expect/actual` when the SDK exists on only some targets.

## One use case per operation

`InitializePayments`, `LoginPayments(userId)`, `LogoutPayments`, `GetOfferings`, `Purchase(package)`, `Restore`, `HasActiveEntitlement`, `ManageSubscription` (store management URL per platform), `ValidateSubscriptionChange` → `Allowed | Upgrade | Downgrade | AlreadySubscribed | Conflict`.

## Identity

- `Purchases.configure(apiKey, appUserId = userId)` after the user is known, or `logIn(userId)` right after configure. Otherwise purchases attach to an anonymous RevenueCat user.
- API key per platform and environment from build config, provided by the app through a `PaymentApiKeyProvider`.
- Run it as a startup `Initializer` after the user initializer (see `feature-startup`).
- Login = auth login, then `logIn`. Logout = **awaited** `logOut()`, then auth logout.

## Purchases

- Products come from **offerings**, never ids hard-coded in a ViewModel.
- **Cancel isn't an error**: check the SDK's `userCancelled` / `PurchasesErrorCode.PurchaseCancelledError`, never the message text. Other errors → a string resource.
- Catch the SDK's own exception around each call and rethrow it as a `ReasonException` with its `ErrorReason`; never `runCatching`, which swallows cancellation.
- Prefer the customer-info listener (`Purchases.sharedInstance.delegate` / updated-customer-info flow) over polling.

## Entitlement is server-side

RevenueCat webhook → your backend → the user row. After a purchase, the UI waits for the new tier on the observed user, with a timeout and a "Restore" fallback. Gate features on that tier, not on the client receipt.

## Paywall screen

`@Serializable data class Paywall(val source: PaywallSource) : Route { override val requiresLogin = true }`, a case of the app's `Route`. Every upsell surface navigates to it; `source` goes into the viewed and purchased analytics events.

## Tests

`ValidateSubscriptionChange` for every tier pair; purchase cancel → no error shown; logout awaits store logout.
