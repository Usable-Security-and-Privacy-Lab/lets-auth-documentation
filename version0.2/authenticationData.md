# Authentication Data

An authenticator encrypts and stores data for the user. This enables the user to
easily use multiple authenticators and replace a lost authenticator.

The authentication data consists of a mapping from the relying party domain to a
set of information needed to authenticate with that domain. There may be more
than one set of information for a given domain if a user has multiple accounts
with that domain. The authenticator encrypts the authentication data in the same
way as a password manager – it uses a symmetric key, then encrypts the symmetric
key with a password-derived key.

## Data Format

For convenience, the authenticator prompts a user to name each authenticator
when it is first initialized. This creates a list of authenticators:

```
authenticatorList = [authenticatorName]
```

To store authentication data, the authenticator creates a JSON map, where the
index for the map is domain of the relying party and the map contains one or
more entries:

```
map(domain) = [entries]
```

where each entry is:

```
entry = [accountName, accountID, sessionPublicKey, sessionPrivateKey]
```

and a `sessionList` is:

```
sessionList = [authenticatorName, [sessionCertificate, geoLocation]]
```

The `geoLocation` is the location of the authenticator when the session
Certificate was generated. Each `sessionCertificate` should be removed from the
list when it expires.

The authenticator encrypts this map with a symmetric key, then encrypts the
symmetric key with a password-derived key:

```
derivedKey = PBKDF(masterPassword)
```

The authentication data is then:

```
authenticationData = {E(derivedKey, symmetricKey) | E(symmetricKey,authenticatorList | map)}
```

## Retrieval

The authenticator should retrieve the authentication data periodically (e.g.
every hour). It may also retrieve it in response to a user explicitly asking to
synchronize the data or by the user navigating to a menu in the authenticator
where the data is expected to be up-to-date. To retrieve the authentication
data, the authenticator uses the following:

### authenticator -> CA

| Method | Path                        | Arguments                |
| ------ | --------------------------- | ------------------------ |
| GET    | /la0.2/users/:username/data | authenticatorCertificate |

The potential responses are:

- 200 OK: success, with the body including the encrypted authentication data
- 403 Forbidden: access denied due to an error validating the authenticator
  certificate
- 404 Not Found: Authentication certificate is valid but no authentication data was found. 
  Should only occur when first called after account registration.

The server response should use the `Etag` header, so that the authenticator
request can use the `If-None-Match` header to validate whether its cached
response is still valid.

## Storage

Because multiple authenticators may be synchronizing the authentication data,
care must be taken to ensure that any updates are atomic. To store data, the
authenticator first obtains a lock:

### authenticator -> CA

| Method | Path                        | Arguments                |
| ------ | --------------------------- | ------------------------ |
| POST   | /la0.2/users/:username/data | authenticatorCertificate |

The potential responses are:

- 200 OK: success, the lock is obtained, and the body contains the encrypted
  authentication data and a random lock identifier
- 403 Forbidden: access denied due to an error validating the authenticator
  certificate
- 409 Conflict: another authenticator has the lock

The lock identifier is a random identifier chosen by the CA that expires after
30 seconds.

After obtaining the lock, the authenticator may update the authentication data:

### authenticator -> CA

| Method | Path                        | Arguments                                                    |
| ------ | --------------------------- | ------------------------------------------------------------ |
| PUT    | /la0.2/users/:username/data | authenticatorCertificate, authenticationData, lockIdentifier |

The potential responses are:

- 200 OK: success, the authentication data is updated
- 403 Forbidden: access denied due to an error validating the authenticator
  certificate
- 409 Conflict: invalid lock identifier
