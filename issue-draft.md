# Only the first AuthenticationMethod of a profile is configured; TTLS-first profiles fail to authenticate on iOS

## Environment

- App: geteduroam iOS app, version: **TODO: fill exact app version**
- iOS: **TODO: fill exact iOS version**
- IdP/profile: Universität Duisburg-Essen (UDE), IdP `5016`, profile `16353`
  (`love2eduroam`)
- Profile methods: PEAP-MSCHAPv2 and TTLS-MSCHAPv2
- Source eap-config:
  `https://cat.eduroam.org/user/API.php?action=downloadInstaller&device=eap-generic&profile=16353`

## Observed behavior

UDE's WLAN team observed the following with the same eduroam CAT profile and
real IdP:

1. Profile order **PEAP-MSCHAPv2 first, TTLS-MSCHAPv2 second**:
   authentication via the geteduroam iOS app works.
2. Profile order **TTLS-MSCHAPv2 first, PEAP-MSCHAPv2 second**:
   authentication via the geteduroam iOS app fails.
3. Reverting the order to PEAP first made the app-based login work again.

The native OS supplicant and the CAT-generated Apple `.mobileconfig` were not
the failing path in this observation; the failure was seen with the geteduroam
iOS app.

Related data point from CAT/mobileconfig: the generated Apple profile appears
to pin only one outer EAP type for this profile:

```xml
<key>AcceptEAPTypes</key>
<array>
  <integer>25</integer>
</array>
```

That is PEAP. The same Apple profile also contains:

```xml
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

So method order is decisive in the generated Apple profile as well. The app
seems to mirror that single-method behavior programmatically.

## Expected behavior

For a profile with multiple compatible username/password authentication
methods, the app should not silently drop all but the first buildable method.
If iOS supports multiple outer EAP types in `NEHotspotEAPSettings`, the app
should either configure the compatible methods together or otherwise document
and report that only one method can be installed.

At minimum, a PEAP + TTLS profile should not become order-sensitive without any
visible indication to the user or IdP operator.

## Actual behavior

Only the first buildable `AuthenticationMethod` is represented in the generated
`NEHotspotEAPSettings`.

In `EAPConfigurator.buildSettings(...)`, the app iterates over
`identityProvider.authenticationMethods.methods`, builds an
`NEHotspotEAPSettings?` for each method, and then selects only the first result:

https://github.com/geteduroam/apple-app/blob/f4b341a89c9e7276f40c1fb83d0f72f0227f6c6d/geteduroam/GeteduroamPackage/Sources/EAPConfigurator/EAPConfigurator.swift#L203-L264

For username/password methods, `buildSettingsWithUsernamePassword(...)` then
writes a single-element `supportedEAPTypes` array:

https://github.com/geteduroam/apple-app/blob/f4b341a89c9e7276f40c1fb83d0f72f0227f6c6d/geteduroam/GeteduroamPackage/Sources/EAPConfigurator/EAPConfigurator.swift#L426-L427

As a result, a PEAP + TTLS profile is effectively reduced to whichever method
appears first and can be built. There is no connect-time fallback to the second
method.

## TTLS inner-auth source encoding

The UDE `.eap-config` source confirms that both PEAP and TTLS use inner
`EAPMethod Type 26`. The TTLS method is not encoded as
`NonEAPAuthMethod Type 3`.

Minimal source excerpt:

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

The app maps TTLS inner authentication methods differently depending on whether
the eap-config encodes MSCHAPv2 as inner EAP or non-EAP:

https://github.com/geteduroam/apple-app/blob/f4b341a89c9e7276f40c1fb83d0f72f0227f6c6d/geteduroam/GeteduroamPackage/Sources/EAPConfigurator/EAPConfigurator.swift#L354-L364

https://github.com/geteduroam/apple-app/blob/f4b341a89c9e7276f40c1fb83d0f72f0227f6c6d/geteduroam/GeteduroamPackage/Sources/EAPConfigurator/EAPConfigurator.swift#L751-L767

- `NonEAPAuthMethod Type 3` is mapped to
  `.eapttlsInnerAuthenticationMSCHAPv2`.
- inner `EAPMethod Type 26` is mapped to
  `.eapttlsInnerAuthenticationEAP`.

Because the UDE TTLS method is encoded as inner `EAPMethod Type 26`, the app
would configure TTLS-EAP-MSCHAPv2 rather than plain TTLS-MSCHAPv2 when TTLS is
the selected outer method. That could explain why the TTLS-first profile fails
against a RADIUS setup expecting plain TTLS-MSCHAPv2, while PEAP-first works
because TTLS is never reached. The exact RADIUS-side rejection mechanism is
still an interpretation of the observed behavior, not a packet-level trace.

## Existing coverage

I could not find existing tests covering this case:

- `ModelsTests.testEntireConfig` decodes one `AuthenticationMethod`, but not a
  PEAP + TTLS multi-method profile.
- `ConnectTests.testValidEAPConfig` also uses one `AuthenticationMethod`.
- I did not find tests for `NonEAPAuthMethod Type 3` vs inner
  `EAPMethod Type 26`.
- I did not find tests asserting how `EAPConfigurator` handles multiple
  username/password methods.

The model type does store methods as an array
(`AuthenticationMethodList.methods: [AuthenticationMethod]`), so this looks
like a configuration-generation issue rather than an XML parsing issue.

## Related issues checked

This does not appear to be a duplicate of the current open issues:

- #154 was reported as "only the first RCOI" for Passpoint, but the current
  code already maps all provider-level OIDs into `roamingConsortiumOIs`.
  A maintainer comment there points at the same `EAPConfigurator.swift`
  `.first` path and says only the first valid method is used. So #154 may be
  another externally visible symptom of the same method-selection limitation.
- #161 is about a second `EAPIdentityProvider` in one `.eap-config`. That is
  another "second entry is not configured" pattern, but at provider level rather
  than multiple `AuthenticationMethod`s inside one `EAPIdentityProvider`.
- #163 and #139 are certificate trust / EAP-TLS issues. This report is about
  username/password PEAP + TTLS method selection; EAP-TLS should remain out of
  scope.
- #83 fixed reading inner non-EAP methods, but this report is about dropping
  the second outer authentication method and the source profile using inner
  `EAPMethod Type 26`.

## Suggested fix direction

For username/password methods only, consider merging compatible methods into one
`NEHotspotEAPSettings` object by writing multiple outer types into
`supportedEAPTypes`.

To keep the change safe, I would not merge methods unless they share the same:

- server-side trust anchors and server names,
- outer identity,
- credential source / username / password semantics,
- non-certificate credential type.

Methods with differing trust anchors, differing server names, differing outer
identities, or client certificates should stay separate. EAP-TLS should not be
changed here.

## Validation offer

UDE can test a fix or TestFlight build against a real IdP with both method
orders:

- PEAP first, TTLS second
- TTLS first, PEAP second

Console logs from `Logger.eap` can also be collected from a failing attempt if
that helps.
