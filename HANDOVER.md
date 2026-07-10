# Handover: geteduroam apple-app - multi-method EAP profile breaks iOS login

Language note: repo, issue and code are English; user-facing communication
with Sören is German.

## Context

Universität Duisburg-Essen (UDE) runs an eduroam CAT profile
(IdP 5016, profile 16353, "love2eduroam") with **two** authentication
methods: PEAP-MSCHAPv2 and TTLS-MSCHAPv2. Real-world observation by the UDE
WLAN team (July 2026):

- Order **PEAP first, TTLS second** -> login via geteduroam iOS app works.
- Order **TTLS first, PEAP second** -> login via geteduroam iOS app FAILS.
- Reverting the order fixed it. Native OS supplicants and the CAT-generated
  `.mobileconfig` were not the failing path; the app was.
- Related data point: the CAT-generated Apple `.mobileconfig` for this
  profile pins only ONE outer EAP type (`AcceptEAPTypes = [25]`, i.e. the
  first method), so ordering is decisive on Apple regardless; the app
  reproduces the same single-method behavior programmatically.

This behavior is NOT currently reported upstream. Closest existing issues:
#161 (second EAPIdentityProvider ignored - different bug), #163 (EAP-TLS
trust). Goal: file a high-quality issue, optionally propose a fix PR.

## Repo and key code locations

Repo: https://github.com/geteduroam/apple-app (Swift, SwiftUI, TCA,
Xcode >= 14.3). Package: `geteduroam/GeteduroamPackage`.

Central file: `Sources/EAPConfigurator/EAPConfigurator.swift`

1. **Single-method selection**: `buildSettings` maps over the profile's
   authentication methods, builds `NEHotspotEAPSettings?` per method, then
   takes `.first` non-nil result. Consequence: only the FIRST buildable method
   is configured; there is no fallback at connect time even though Apple's
   `NEHotspotEAPSettings.supportedEAPTypes` is an **array** and could carry
   multiple outer types.

2. **Per-method settings**: for username/password methods the inner auth type
   is derived from `innerAuthenticationMethods` via
   `compactMap { ... }.first ?? .eapttlsInnerAuthenticationMSCHAPv2`, then
   `buildSettingsWithUsernamePassword` sets `supportedEAPTypes =
   [outerEapType]` (single element) and `ttlsInnerAuthenticationType =
   innerAuthType` (also set for PEAP, where iOS likely ignores it).

3. **Inner-auth mapping**: `getInnerAuthMethod` maps eap-config inner methods
   to `TTLSInnerAuthenticationType`. Suspected failure mode: if the profile
   encodes TTLS inner auth as **EAP-MSCHAPv2 (`<EAPMethod><Type>26`)**,
   mapping yields `.eapttlsInnerAuthenticationEAP` -> on the wire this is
   **TTLS-EAP-MSCHAPv2**, which RADIUS servers configured for plain
   **TTLS-MSCHAPv2** may reject. If encoded as
   **`<NonEAPAuthMethod><Type>3`** (non-EAP MSCHAPv2, mapped via the negated
   rawValue trick) it yields `.eapttlsInnerAuthenticationMSCHAPv2` and should
   work.

## Open input needed from Sören

- [ ] The TTLS `<AuthenticationMethod>` block of UDE's `.eap-config`
      (to confirm whether inner auth is `EAPMethod Type 26` vs
      `NonEAPAuthMethod Type 3`). Sören has the file locally.
- [ ] iOS version and app version used in the failing test. Console.app logs
      for `Logger.eap` from a failing attempt would be ideal.

## Tasks

### T1 - Verify hypotheses in code

- Read `EAPConfigurator.swift` fully; confirm the `.first` selection path and
  the inner-auth mapping table with exact line numbers.
- Check `Tests/ConnectTests` and `Tests/ModelsTests` for existing coverage of
  multi-method profiles and TTLS inner-auth parsing; note gaps.
- Check whether an eap-config with two `AuthenticationMethod`s parses both
  (Models layer) - i.e. whether the loss happens in EAPConfigurator, not
  parsing.

### T2 - Draft the GitHub issue

Markdown, English, ready for Sören to paste. Structure:

- Title suggestion: "Only the first AuthenticationMethod of a profile is
  configured; TTLS-first profiles fail to authenticate on iOS"
- Environment (app version, iOS version - placeholders), IdP/profile id.
- Reproduction (the UDE order-swap observation, both directions).
- Analysis: the `.first` selection with code permalink(s); the
  supportedEAPTypes single-element construction; the inner-auth mapping
  question with the eap-config snippet placeholder until Sören supplies it.
- Expected vs actual behavior.
- Offer: UDE can test against a real IdP with both orderings, including an
  upcoming release/TestFlight.
- Keep speculation clearly labeled as hypothesis, especially the
  TTLS-EAP-MSCHAPv2 point.

### T3 - Optional fix PR

Only after T1 confirms. Smallest safe change: when multiple username/password
methods exist that share the same `ServerSideCredential` (CAs + server names)
and the same `outerIdentity`, merge their outer EAP types into one
`supportedEAPTypes` array instead of dropping all but the first. Do **not**
merge across methods with differing trust anchors, server names, outer
identities, or credential types. EAP-TLS stays separate.

Add unit tests for: two-method merge, differing-trust no-merge, TTLS inner-auth
mapping for both encodings (`EAPMethod Type 26` vs `NonEAPAuthMethod Type 3`).

## Constraints and pitfalls

- `NetworkExtension` / `NEHotspotConfiguration` compiles only against Apple
  SDKs. If the sandbox has no usable macOS/Xcode toolchain, limit to static
  analysis plus test code that Sören compiles locally; state this honestly.
- Follow existing TCA/module structure; no drive-by refactors.
- License: check repo license before submitting. Contribution will be made
  under Sören's GitHub account.
- Do not touch the EAP-TLS path (issue #163 territory).
- All commits/PR text in English.

## Done criteria

1. Issue draft exists as `issue-draft.md`, hypotheses labeled, placeholders for
   eap-config snippet and versions clearly marked.
2. T1 findings summarized in `analysis-notes.md` with exact code refs.
3. Optional PR branch with merge logic and tests, compiling status honestly
   documented.
