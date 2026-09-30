# itb-plugin-jwt

JSON Web Token vocabulary for the OTB Gherkin language: decode a compact JWS and
assert on its claims.

Dialect only. This plugin deploys nothing.

## Use it

Declare the dialect in the feature's tags. It needs a `fhir-validator` actor in
scope — see **How it works** for why:

```gherkin
@lang:itb-core-en@^2 @dialect:fhir-validator@^2 @dialect:jwt@^1
```

```gherkin
Background:
  Given AuthorizationServer is the system under test at "https://as.example.org"
  And FHIRValidator is a fhir-validator at "http://fhir-validator:8080"
  And Client is infrastructure

Scenario: the issued token carries the claims the profile requires
  Given Client is asked for $jwt with "Paste an access token"
  Then $jwt should be a well-formed JWT
  When Client decodes the JWT $jwt as $claims
  Then $claims at "/iss" should not be empty
  And $claims at "/aud" should contain "https://rs.example.org"
  And $claims at "/exp" should not be empty
```

The claim set is an ordinary value, so every path assertion in the core language
works on it: `at "/iss" should be`, `should contain`, `should not exist`, and so
on.

## The verbs

```gherkin
{actor} decodes the JWT {ref}( on {actor:fhir-validator})? as {var}
{actor} reads the JWT header of {ref}( on {actor:fhir-validator})? as {var}
{ref} should be a well-formed JWT( on {actor:fhir-validator})?
```

The `on <Validator>` slot is optional; leave it off and it resolves to the one
declared `fhir-validator`. There is no actor kind of its own here — actor kinds
are a global namespace, so referring to another dialect's kind is legitimate.

`should be a well-formed JWT` checks two things, because the first alone is not
enough: three dot-separated segments, **and** a claim segment that actually
decodes.

## How it works, and what that costs you

Decoding a compact JWS is base64url plus a JSON parse, and GITB TDL has no
base64 processor. Rather than stand up a service for it, these verbs borrow the
FHIR validator's FHIRPath endpoint:

```
POST /itb/fhirPath/process   { operation: evaluate, input: [content, expression] }
  -> /output/0/value
```

with expressions of the form

```
'<jws>'.split('.')[1].decode('urlbase64')
```

Verified against `validator_cli server -version 5.0`: `split()` returns the
segment, `decode('urlbase64')` returns the claims as JSON, and
`split('.').count()` gives the segment count.

**The caveat, stated because it decides who owns a future break.** Neither
`split()` nor `decode()` appears in the FHIRPath N1 specification or in FHIR R5's
list of added functions. They exist in the HL7 Java implementation. This dialect
therefore rests on an implementation detail rather than on a standard. If a
validator release drops them, these verbs break, and the fix is a small service
of our own rather than a change here.

## Limits

**No signature verification.** Decoding a token is not validating it. Nothing
here checks the signature against the issuer's keys, and a forged token decodes
exactly as well as a genuine one. If your test needs to establish that a token
was really issued by a particular server, this dialect cannot do it.

**No token acquisition.** Getting a token in the first place is a separate
concern; use [`itb-plugin-oauth`](https://github.com/OpenTestBed/itb-plugin-oauth)
alongside this one.

## Who uses it

IHE IUA, SMART on FHIR, UDAP — any profile that carries authorization in a
bearer token. The IUA suite in `itb-plugin-authoring` is the worked example.

## Licence

BSD-3-Clause.
