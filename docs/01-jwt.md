# Working with JWT

In this document we describe how to work with JWT using `github.com/lestrrat-go/jwx/jwt`

* [Terminology](#terminology)
  * [Verification](#verification)
  * [Validation](#validation)
* [Parsing](#parsing)
  * [Parse a JWT](#parse-a-jwt)
  * [Parse a JWT from file](#parse-a-jwt-from-file)
  * [Parse a JWT from a *http.Request](#parse-a-jwt-from-a-httprequest)
  * [Parse a private claim into private struct](#parse-a-private-claim-into-a-custom-struct)
* [Verification](#jwt-verification)
  * [Parse and Verify a JWT (with a single key)](#parse-and-verify-a-jwt-with-single-key)
  * [Parse and Verify a JWT (with a key set, matching "kid")](#parse-and-verify-a-jwt-with-a-key-set-matching-kid)
* [Validation](#jwt-validation)


---

# Terminology

## Verification

We use the terms "verify" and "verification" to describe the process of ensuring the integrity of the JWT, namely the signature verification.

## Validation

We use the terms "validate" and "validation" to describe the process of checking the contents of a JWT, for example if the values in fields such as "iss", "sub", "aud" match our expected values, and/or if the token has expired.

# Parsing

## Parse a JWT

To parse a JWT in either raw JSON or JWS compact serialization format, use [`jwt.Parse()`](https://pkg.go.dev/github.com/lestrrat-go/jwx/jwt#Parse)

```go
src := []byte{...}
token, _ := jwt.Parse(src)
```

Note that the above form does NOT perform any signature verification, or validation of the JWT token itself.
This just reads the contents of src, and maps it into the token, period.
In order to perform verification/validation, please see the methods described elsewhere in this document, and pass the appropriate option(s).

## Parse a JWT from file

To parsea JWT stored in a file, use [`jwt.ReadFile()`](https://pkg.go.dev/github.com/lestrrat-go/jwx/jwt#ReadFile). [`jwt.ReadFile()`](https://pkg.go.dev/github.com/lestrrat-go/jwx/jwt#ReadFile) accepts the same options as [`jwt.Parse()`](https://pkg.go.dev/github.com/lestrrat-go/jwx/jwt#Parse).

```go
token, _ := jwt.ReadFile(`token.json`)
```

## Parse a JWT from a *http.Request

To parse a JWT stored within a *http.Request object, use [`jwt.ParseRequest()`](https://pkg.go.dev/github.com/lestrrat-go/jwx/jwt#ParseRequest). It by default looks for JWTs stored in the "Authorization" header, but can be configured to look under other headers and within the form fields.

```go
// Looks under "Authorization" header
token, err := jwt.ParseRequest(req)

// Looks under "X-JWT-Token" header
token, err := jwt.ParseRequest(req, jwt.WithHeaderKey("X-JWT-Token")

// Looks under "Authorization" and "X-JWT-Token" headers
token, err := jwt.ParseRequest(req, jwt.WithHeaderKey("Authorization"), jwt.WithFormKey("X-JWT-Token"))

// Looks under "Authorization" header and "access_token" form field
token, err := jwt.ParseRequest(req, jwt.WithFormKey("access_token"))
```

## Parse a Private Claim into a Custom Struct

By default all JWT private claims are parsed into whatever `"encoding/json".Unmarshal` deems appropriate. For example, JSON objects are turned into `map[string]interface{}`, JSON arrays are turned into `[]interface{}`, etc.

In case you want to parse these into a struct of your choice, you have two options.

The first option is to register a mapping from a private claim field name to a struct:

```go
const src = `{
  ...
  "x-my-custom-claim": {
    "foo": 1,
    "bar": true
  }
}`

type MyCustomClaim struct {
  Foo int `json:"foo"`
  Bar bool `json:"bar"`
}

jwt.RegisterCustomField(`x-my-custom-claim`, MyCustomClaim{})

token, _ := jwt.Parse([]byte(src))
v, _ := token.Get(`x-my-custom-claim`)
mcc := v.(MyCustomClaim)
fmt.Printf("foo is %d, bar is %t\n", mcc.Foo, mcc.Bar)
```

This has a global effect. All top-level JWT claims with the name "x-my-custom-claim" will then be parsed into an object of type `MyCustomClaim`. This is usually the preferred way, as it leaves no ambiguities.

But sometimes you end up with an application that has separate JWTs with overlapping field names.
In such cases, you may opt to use a per-call custom claim:

```go
token, _ := jwt.Parse([]byte(src), jwt.WithTypedClaim(`x-my-custom-claim`, MyCustomClaim{}))
v, _ := token.Get(`x-my-custom-claim`)
mcc := v.(MyCustomClaim)
fmt.Printf("foo is %d, bar is %t\n", mcc.Foo, mcc.Bar)
```

This has local effect, and overrides any global settings. The downside is that now each `jwt.Token` instance must be carefully managed so that you don't mix up tokens that were parsed with this option and those that weren't.

# JWT Verification

## Parse and Verify a JWT (with single key)

To parse a JWT *and* verify that its content matches the signature as described in the JWS message, you need to add some options when calling the [`jwt.Parse()`](https://pkg.go.dev/github.com/lestrrat-go/jwx/jwt#Parse) function. Let's assume the signature was generated using ES256:

```go
src := []byte{...}
token, _ := jwt.Parse(src, jwt.WithVerify(jwa.ES256, key))
```

In the above example, `key` may either be the raw key (i.e. "crypto/ecdsa".PublicKey, "crypto/ecdsa".PrivateKey) or an instance of [`jwk.Key`](https://pkg.go.dev/github.com/lestrrat-go/jwx/jwk#Key) (i.e. [`jwk.ECDSAPrivateKey`](https://pkg.go.dev/github.com/lestrrat-go/jwx/jwk#ECDSAPrivateKey), [`jwk.ECDSAPublicKey`](https://pkg.go.dev/github.com/lestrrat-go/jwx/jwk#ECDSAPublicKey)). The key type must match the algorithm being used.

## Parse and Verify a JWT (with a key set, matching "kid")

To parse a JWT *and* verify that its content matches the signature as described in the JWS message using a [`jwk.Set`](https://pkg.go.dev/github.com/lestrrat-go/jwx/jwk#Set), you need to add some options when calling the [`jwt.Parse()`](https://pkg.go.dev/github.com/lestrrat-go/jwx/jwt#Parse) function. Let's assume the JWS contains the "kid" header of the key that generated the signature:

```go
src := []byte{...}
token, _ := jwt.Parse(src, jwt.WithKeySet(keyset))
```

The above example will correctly verify the message if the jwk.Set specified by the variable `keyset` contains a key that matches
the key ID in the JWS message.

# JWT Validation

To validate if the JWT's contents, such as if the JWT contains the proper "iss","sub","aut", etc, or the expiration information and such, use the [`jwt.Validate()`](https://pkg.go.dev/github.com/lestrrat-go/jwx/jwt#Validate) function.

```go
if err := jwt.Validate(token); err != nil {
	return errors.New(`failed to validate token`)
}
```

By default we only check for the time-related components of a token, such as "iat", "exp", and "nbf". To tell [`jwt.Validate()`](https://pkg.go.dev/github.com/lestrrat-go/jwx/jwt#Validate) to check for other fields, use one of the various [`jwt.ValidateOption`](https://pkg.go.dev/github.com/lestrrat-go/jwx/jwt#ValidateOption) values.

```go
// Check for token["iss"] == "github.com/lestrrat-go/jwx"
if err := jwt.Validate(token, jwt.WithIssuer(`github.com/lestrrat-go/jwx`)) {
  return errors.New(`failed to validate token`)
}
```
