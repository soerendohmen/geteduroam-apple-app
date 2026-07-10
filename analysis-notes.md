# Analysis notes: multi-method EAP profile handling

Date: 2026-07-10

Repository state inspected: `geteduroam/apple-app` at
`f4b341a89c9e7276f40c1fb83d0f72f0227f6c6d`, local fork remote
`soerendohmen/geteduroam-apple-app`.

## T1 findings

### 1. Only one buildable AuthenticationMethod is configured

Confirmed.

`EAPConfigurator.buildSettings(...)` iterates over
`identityProvider.authenticationMethods.methods`, attempts to build one
`NEHotspotEAPSettings?` per `AuthenticationMethod`, filters unusable methods
with `compactMap`, then immediately takes `.first`.

Code refs:

- `geteduroam/GeteduroamPackage/Sources/EAPConfigurator/EAPConfigurator.swift:197`
  starts `buildSettings(...)`.
- `.../EAPConfigurator.swift:203-206` iterates
  `identityProvider.authenticationMethods.methods.compactMap`.
- `.../EAPConfigurator.swift:262-264` returns each built setting and selects
  `.first`.

Implication: if a profile contains PEAP and TTLS, the app configures only the
first method that survives validation and certificate import. The second method
is not represented in the resulting `NEHotspotEAPSettings`, so iOS cannot fall
back to it during association.

### 2. Username/password EAP settings carry a single supported outer EAP type

Confirmed.

For `.EAPTTLS`, `.EAPFAST`, and `.EAPPEAP`, the per-method builder derives one
outer EAP type from `authenticationMethod.EAPMethod.type`, derives one TTLS
inner authentication type from the first parseable inner method, and then calls
`buildSettingsWithUsernamePassword(...)`.

Code refs:

- `.../EAPConfigurator.swift:305` enters the username/password path for
  `.EAPTTLS`, `.EAPFAST`, `.EAPPEAP`.
- `.../EAPConfigurator.swift:353-364` derives `outerIdentity` and the first
  supported inner auth type, defaulting to MSCHAPv2.
- `.../EAPConfigurator.swift:426` sets
  `eapSettings.supportedEAPTypes = [NSNumber(value: outerEapType.rawValue)]`.
- `.../EAPConfigurator.swift:427` sets
  `eapSettings.ttlsInnerAuthenticationType = innerAuthType`.

Implication: even though `NEHotspotEAPSettings.supportedEAPTypes` is an array,
this path always writes a single-element array for username/password methods.

### 3. TTLS inner-auth mapping distinguishes EAP-MSCHAPv2 from non-EAP MSCHAPv2

Confirmed.

`buildSettings(...)` maps inner `EAPMethod` values directly and maps
`NonEAPAuthMethod` values by negating the raw value before calling
`getInnerAuthMethod(...)`.

Code refs:

- `.../EAPConfigurator.swift:354-364` maps inner methods and chooses the first
  supported one.
- `.../EAPConfigurator.swift:751-767` defines `getInnerAuthMethod(...)`.
- `.../EAPConfigurator.swift:757-758` maps `-3` (Non-EAP MSCHAPv2) to
  `.eapttlsInnerAuthenticationMSCHAPv2`.
- `.../EAPConfigurator.swift:763-764` maps `26` (EAP-MSCHAPv2) to
  `.eapttlsInnerAuthenticationEAP`.

Implication: the UDE TTLS failure has two layers that should stay separate in
the issue:

- confirmed: method order controls which single outer EAP type the app
  configures;
- confirmed from UDE `.eap-config`: the TTLS method is encoded as inner
  EAP-MSCHAPv2 (`Type 26`) rather than non-EAP MSCHAPv2 (`Type 3`), so the
  current app mapping selects `.eapttlsInnerAuthenticationEAP`. The remaining
  hypothesis is whether that exact TTLS-EAP-MSCHAPv2 behavior is what the UDE
  RADIUS side rejects.

### 3b. UDE eap-config source

The source eap-config is available from the CAT generic EAP download endpoint:

`https://cat.eduroam.org/user/API.php?action=downloadInstaller&device=eap-generic&profile=16353`

The response headers identify it as:

```text
content-type: application/eap-config
content-disposition: inline; filename="eduroam-eap-generic-UoD-love2eduroam.eap-config"
```

Minimal relevant source excerpt:

```xml
<AuthenticationMethod>
  <EAPMethod>
    <Type>25</Type>
  </EAPMethod>
  ...
  <InnerAuthenticationMethod>
    <EAPMethod>
      <Type>26</Type>
    </EAPMethod>
  </InnerAuthenticationMethod>
</AuthenticationMethod>
<AuthenticationMethod>
  <EAPMethod>
    <Type>21</Type>
  </EAPMethod>
  ...
  <InnerAuthenticationMethod>
    <EAPMethod>
      <Type>26</Type>
    </EAPMethod>
  </InnerAuthenticationMethod>
</AuthenticationMethod>
```

So the UDE source profile has PEAP first (`25`), TTLS second (`21`), and the
TTLS method uses inner EAP-MSCHAPv2 (`Type 26`), not
`NonEAPAuthMethod Type 3`.

### 3c. UDE Apple mobileconfig cross-check

Sören provided the CAT-generated Apple profile:

`/Users/sorendohmen/Downloads/eduroam-OS_X-Universitat_Duisburg-Essen-love2eduroam.mobileconfig`

It is a signed/binary-ish profile (`file` reports `data`; `plutil -p` cannot
parse it directly), but the embedded plist content is visible with `strings`.
The Wi-Fi payload contains:

```xml
<key>AcceptEAPTypes</key>
<array>
  <integer>25</integer>
</array>
...
<key>OuterIdentity</key>
<string>eduroam@uni-due.de</string>
...
<key>TLSTrustedServerNames</key>
<array>
  <string>radius1.uni-duisburg-essen.de</string>
  <string>radius2.uni-duisburg-essen.de</string>
</array>
...
<key>TTLSInnerAuthentication</key>
<string>MSCHAPv2</string>
```

This confirms the Apple profile generated by CAT is single-outer-EAP and pins
PEAP (`25`) for the provided ordering. It also confirms the Apple profile's
TTLS inner-auth value is plain `MSCHAPv2`.

It does **not** answer the remaining eap-config question, because the
`.mobileconfig` is the converted Apple output and no longer contains the source
`<AuthenticationMethod>` XML encoding that would distinguish
`<EAPMethod><Type>26</Type>` from `<NonEAPAuthMethod><Type>3</Type>`.

The eap-config source above now answers that question: source TTLS is
`EAPMethod Type 26`.

### 4. Models layer stores AuthenticationMethod as an array

Confirmed statically.

`AuthenticationMethodList` defines `methods: [AuthenticationMethod]` and maps
the XML key `AuthenticationMethod` to that array.

Code refs:

- `geteduroam/GeteduroamPackage/Sources/Models/EAP/AuthenticationMethodList.swift:4-13`.

This strongly suggests the multi-method loss occurs in `EAPConfigurator`, not
in the model type. However, there is no existing unit test that decodes two
`AuthenticationMethod` siblings and asserts both are retained.

### 5. Existing test coverage gaps

Observed tests:

- `geteduroam/GeteduroamPackage/Tests/ModelsTests/ModelsTests.swift:70-147`
  decodes a full eap-config, but it contains only one `AuthenticationMethod`
  and that method is EAP-TLS (`Type 13`).
- `geteduroam/GeteduroamPackage/Tests/ConnectTests/ConnectTests.swift` has
  valid/invalid eap-config flow tests; the valid sample also contains only one
  `AuthenticationMethod`, again EAP-TLS (`Type 13`).
- No test currently covers PEAP + TTLS in one profile.
- No test currently covers `NonEAPAuthMethod Type 3` vs inner
  `EAPMethod Type 26`.
- No direct unit test currently covers `EAPConfigurator` behavior for multiple
  username/password authentication methods.

### 6. Local verification status

Command attempted:

```sh
swift test --package-path geteduroam/GeteduroamPackage --filter ModelsTests
```

First run in the sandbox failed before manifest loading because Swift/Clang
could not write their normal caches under the user home.

Second run outside the sandbox fetched and resolved dependencies, started
building, and compiled the `Models` target. It then failed before running
`ModelsTests` while building unrelated package targets:

```text
Sources/AuthClient/OIDAuthState.swift:25:94: error: type 'Bundle' has no member 'module'
```

So the local result is: static analysis completed; SwiftPM test execution is
blocked by the current package build setup in this environment, not by the
multi-method analysis itself.

### 7. Related upstream issues checked

GitHub issues were checked through the public API on 2026-07-10.

Relevant but not duplicates:

- `#161` (open): second `EAPIdentityProvider` in one `.eap-config` is not
  configured. Related "second thing ignored" pattern, but different XML level:
  provider-level, not multiple `AuthenticationMethod`s inside one provider.
- `#154` (open): reported as only first RCOI configured for Passpoint. The
  current code already maps all provider-level OIDs into
  `hs20.roamingConsortiumOIs`, but a maintainer comment on 2025-04-08 says
  "Indeed only the first valid method is used" and links to the same
  `EAPConfigurator.swift` `.first` selection path. This is likely another
  externally visible symptom of the same method-selection limitation, although
  the user-facing symptom is Passpoint/RCOI rather than PEAP-vs-TTLS.
- `#163` (open): EAP-TLS certificate trust issue where app-installed profile
  fails but manual `.mobileconfig` works. Different credential type and trust
  path; EAP-TLS should stay out of our proposed fix.
- `#139` (open): CA rotation / trust store problem. Certificate trust class,
  not multi-method EAP selection.
- `#83` (closed PR): fixed inner non-EAP method not being read. This is
  historically relevant because our issue depends on the distinction between
  `NonEAPAuthMethod Type 3` and `EAPMethod Type 26`, but it does not cover
  dropping the second outer authentication method.
- `#122` (closed): "No valid outer EAP type"; adjacent EAP configuration
  error, not this order-sensitive multi-method failure.

Conclusion: file a new issue. It should explicitly mention `#154` as likely
sharing the same `.first valid method` root cause, `#161` as a similar
provider-level limitation, and `#163`/`#139` as distinct trust/EAP-TLS issues.

### 8. Broader `.first` / first-entry audit

Searched the package sources for `.first`, `first(where:)`, `firstObject`,
`prefix(1)`, `supportedEAPTypes`, `authenticationMethods`, `providers`,
`IEEE80211`, and related eap-config model fields.

Potentially relevant configuration truncation points:

- `ConnectFeature.swift:1023-1034`: after decoding `EAPIdentityProviderList`,
  the app chooses the first valid provider:
  `providerList.providers.first(where: validUntil...)`. This is the code path
  that aligns with issue `#161`: multiple `EAPIdentityProvider` entries can be
  decoded, but only one is passed into `EAPConfigurator`.
- `EAPConfigurator.swift:203-264`: inside the chosen provider, the app maps
  all `AuthenticationMethod`s to possible `NEHotspotEAPSettings`, then keeps
  only `.first`. This is the exact path for the UDE PEAP/TTLS issue and likely
  also the same root cause behind the maintainer comment on `#154` ("only the
  first valid method is used").
- `EAPConfigurator.swift:354-364`: inside one authentication method, the app
  maps all `InnerAuthenticationMethod`s and keeps the first supported one,
  defaulting to MSCHAPv2. This can be legitimate if Apple's API accepts only
  one `ttlsInnerAuthenticationType`, but it is another order-sensitive choice.
  It matters for the UDE source profile because `EAPMethod Type 26` maps to
  `.eapttlsInnerAuthenticationEAP`.

Checked but probably not the same bug:

- `EAPConfigurator.swift:145-180`: all `CredentialApplicability.IEEE80211`
  `ConsortiumOID` values are collected with `compactMap`, uppercased with
  `oids.map`, and assigned to `hs20.roamingConsortiumOIs`. The current code
  does not directly take only the first RCOI. This is why `#154` is more likely
  about first valid **method** selection than first OID selection.
- `EAPConfigurator.swift:152-184`: all SSIDs are collected and a separate
  `NEHotspotConfiguration` is appended for each SSID.
- `EAPConfigurator.swift:209-239` and `513-532`: server IDs and CA
  certificates are handled as arrays; CA import appends all successfully
  imported certs.
- `EAPConfigurator.swift:634-639`: `SecPKCS12Import` uses `items.firstObject`
  and logs if multiple identities are present. This is an EAP-TLS/client-cert
  special case and should stay out of the PEAP/TTLS fix.
- `ConnectFeature.swift:144-148` and `628`: profile selection uses
  `first(where:)` to select the explicit or default UI profile. This is normal
  UI selection, not silent loss inside one eap-config profile.
- `LocalizedEntry.localized(...)` and `LocalizedString.localized(...)`: use
  first matching language or fallback entry. This is normal localization
  fallback behavior.
- `NotificationClient.swift:162`: reads one pending renewal reminder from two
  known notification identifiers. Not related to EAP profile generation.
- `GeteduroamAppDelegate.swift:86-90`: selects current window scene/key window.
  UI plumbing only.

Overall: the strongest shared code-level issue is not "every `.first` is bad";
it is specifically that the eap-config hierarchy is reduced at two levels:
first valid `EAPIdentityProvider`, then first valid `AuthenticationMethod`.
The UDE report should focus on the second level while acknowledging that `#161`
points at the first level and `#154` likely surfaced the same second-level
method-selection limit through Passpoint behavior.

## T3 notes if a fix PR is attempted

Potentially safe direction:

- Keep EAP-TLS separate.
- Only consider merging `.EAPTTLS`, `.EAPPEAP`, and maybe `.EAPFAST` methods
  that use username/password credentials.
- Merge only when methods have the same server-side trust configuration
  (same CA material and same server IDs), same outer identity, same username,
  and same password source.
- Do not merge methods with different trust anchors, different server names,
  different outer identities, different credential types, or any client
  certificate requirement.
- Preserve method order when writing `supportedEAPTypes`.

Open design question for a PR:

- `NEHotspotEAPSettings` has only one `ttlsInnerAuthenticationType` property.
  If PEAP + TTLS are merged into one settings object, this property can still
  represent the TTLS inner auth choice, while PEAP likely ignores it. Tests and
  real-device validation should focus on this exact assumption.

## Senior/auditor re-review

Re-reviewed the draft and evidence with a stricter upstream-readiness lens.

Findings:

- Strong evidence: the app code definitely reduces multiple
  `AuthenticationMethod`s to the first buildable settings object.
- Strong evidence: the UDE source eap-config definitely contains PEAP (`25`)
  and TTLS (`21`) methods, both with inner `EAPMethod Type 26`.
- Strong evidence: the generated Apple profile for the PEAP-first order is
  single-method and pins `AcceptEAPTypes = [25]`.
- Moderate inference: TTLS-first fails because the app configures
  TTLS-EAP-MSCHAPv2 and UDE RADIUS expects plain TTLS-MSCHAPv2. This is
  consistent with code and observation, but remains an inference until a
  packet/RADIUS trace or app log confirms the exact rejection.
- Draft risk reduced: the issue text now treats `.mobileconfig` as supporting
  context rather than proof of app behavior, and distinguishes the
  operator-level "TTLS-MSCHAPv2" description from the source eap-config's
  `EAPMethod Type 26` encoding.

Senior recommendation:

- File the issue before attempting a fix PR.
- Phrase the core bug as "the app silently keeps only the first valid
  AuthenticationMethod" rather than "TTLS is wrong" or "RCOI is wrong".
- Mention `#154` as likely same `.first valid method` root cause, not merely
  vaguely related.
- Keep `#161` as a sibling design limitation one level higher in the XML tree.
- Do not submit a code fix that merges methods across different trust anchors,
  outer identities, credential types, or EAP-TLS.
