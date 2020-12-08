# Account Recovery

An authenticator encrypts and stores recovery data for the user. This consists
of a mapping from services to a set of information that includes the secret for
that service, the current service certificates, and the associated authenticator
device names. The authenticator encrypts the recovery data in the same way as a
password manager – it uses a symmetric key, then encrypts the symmetric key with
a password-derived key.

## Data Format

To store recovery data, the authenticator creates a JSON map, where the index
for the map is domain of the service and the map contains:

- a list of `deviceName` / `serviceCertificate` pairs for that service
- the `serviceSecret` for that service

The authenticator encrypts this map with a symmetric key, then encrypts the
symmetric key with a password-derived key:

```
derivedKey = PBKDF(masterPassword)
```

The recovery data is then:

```
recoveryData = {E(derivedKey, symmetricKey) | E(symmetricKey,map)}
```

## Retrieval

The authenticator should retrieve the recovery data periodically (e.g. every
hour). It may also retrieve it in response to a user explicitly asking to
synchronize the data or by the user navigating to a menu in the authenticator
where the data is expected to be up-to-date. To retrieve the recovery data, the
authenticator uses the following:

### authenticator -> CA

| Method | Path                            | Arguments                |
| ------ | ------------------------------- | ------------------------ |
| GET    | /la0.2/users/:username/recovery | authenticatorCertificate |

The potential responses are:

- 200 OK: success, with the body including the encrypted recovery data
- 403 Forbidden: access denied due to an error validating the authenticator
  certificate

The server response should use the `Etag` header, so that the authenticator
request can use the `If-None-Match` header to validate whether its cached
response is still valid.

## Storage

Because multiple authenticators may be synchronizing the recovery data, care
must be taken to ensure that any updates are atomic. To store data, the
authenticator first obtains a lock:

### authenticator -> CA

| Method | Path                            | Arguments                |
| ------ | ------------------------------- | ------------------------ |
| POST   | /la0.2/users/:username/recovery | authenticatorCertificate |

The potential responses are:

- 200 OK: success, the lock is obtained, and the body contains the encrypted
  recovery data and a random lock identifier
- 403 Forbidden: access denied due to an error validating the authenticator
  certificate
- 409 Conflict: another authenticator has the lock

The lock identifier is a random identifier chosen by the CA that expires after
30 seconds.

After obtaining the lock, the authenticator may update the recovery data:

### authenticator -> CA

| Method | Path                            | Arguments                                              |
| ------ | ------------------------------- | ------------------------------------------------------ |
| PUT    | /la0.2/users/:username/recovery | authenticatorCertificate, recoveryData, lockIdentifier |

The potential responses are:

- 200 OK: success, the recovery data is updated
- 403 Forbidden: access denied due to an error validating the authenticator
  certificate
- 409 Conflict: invalid lock identifier
