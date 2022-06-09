# Service Authentication

Service authentication begins with a client requesting entry (registration or
login) via 1Key with a web service or app.

## Client Request

To authenticate to a web service, the client requests a session object from the
web server:

### client -> web service

| Method | Path           | Arguments |
| ------ | -------------- | --------- |
| GET    | /api/sessionID |           |

The potential responses are:

- 200 OK: success, with the body containing sessionID, a unique session ID for
this client

## Authenticator Transfer

The client device relays the `sessionID` and the `domain` + /api/login to the
authenticator via:

- presenting it as a QR code for scanning by a smartphone authenticator
- passing it via a uri scheme from the client smartphone app to the 1Key
smartphone app
- providing the information in hidden form fields for a browser extension
  authenticator
- providing the information in a deep link for a desktop authenticator

## Site Certificate

Once the authenticator validates the signed object it checks whether it has a
valid _site certificate_ for the desired service. If it does not, it obtains
a site certificate from the CA. To do this, it creates a new service key
pair, `sitePrivateKey` and `sitePublicKey`. It checks if the domain of the
service has ever been registered for this user. If not, it generates a random
identifier, `uid`.  Otherwise, it reuses the existing `uid` for this web service.

The authenticator signs the `sessionID` using the `sitePrivateKey`

The authenticator then generates a CSR, or requestedCert:

```
requestedCert = certificateSigningRequest(uid,sitePublicKey)
```

that is signed by the `authenticatorPrivateKey`.

### authenticator -> CA

| Method | Path            | Arguments                                                                                    |
| ------ | --------------- | -------------------------------------------------------------------------------------------- |
| POST   | /users/cert/csr | username, deviceCert, deviceName, requestedCert, csrSignature, goodUntil, savedData, address |

where

- csrSignature is a signature of the requestedCert with the
`authenticatorPrivateKey`, so that the CA can verify the authenticator is
authorized to claim ownership of the `uid`.

The potential responses are:

- 200 OK : success, with the body containing siteCertificate (csr)
- 400 Bad Request: denied for invalid signature or failed to sign `requestedCert`
- 500 Internal Server Error: CA failed to save newly signed `requestedCert`

The `siteCertificate` contains (uid, sitePublicKey) and is signed by
the `caPrivateKey`.

## Registration and Login

Whether the authenticator is registering a new account with the web service
or just logging in, it sends the login POST request to the webservice:

### authenticator -> service

| Method | Path       | Arguments              |
| ------ | ---------- | ---------------------- |
| POST   | /api/login | cert, signedUuid, uuid |

where:
- `cert` is the same as `siteCertificate` obtained from the CA
- `signedUuid` is the `sessionID` that has been signed by the `sitePrivateKey`
- `uuid` is the same as the original `sessionID`

The potential responses are:

- 200 OK : success, even if user needed registration
- 400 Bad Request: invalid sessionID
- 401 Unauthorized: cert has been revoked
- 500 Internal Server Error: CA fails to redirect user for registration or fails
to save cert

The web service validates the `signedUuid` using the `sitePublicKey` and ensures
the `uuid` (`sessionID`) is one it recently issued.
Then the service validates the `siteCertificate` using the `CAPublicKey`. It
checks if the user needs registration by identifying the user with the `uid`
from `siteCertificate`. If no user is registered with the service, the service
redirects the client to registration, otherwise the user successfully completes
login.

After redirecting for registration, the client prompts the user for `firstName`
and `lastName` and calls the registration POST request to the webservice:

### client -> service

| Method | Path          | Arguments                                   |
| ------ | ------------- | ------------------------------------------- |
| POST   | /api/register | sessionID, certificate, firstName, lastName |

where:
- `sessionID` is the same as `uuid` from the login endpoint
- `certificate` is the same as `siteCertificate` from the login endpoint

The potential responses are:

- 200 OK : success registering user
- 400 Bad Request: sessionID not associated with any certificates or bad certificate
- 500 Internal Server Error: CA fails to register user or login once registered

The service stores the `uid` from `certificate` as the userID for the new
account.

## Notifying the client

For login, the client needs to be notified whether to redirect to Registration
or if login was completed. This is accomplished through a websocket with the
service.
When opened, the client creates a websocket connection with the service at the
path /users/cert/websocket.
When the client request is initiated, the client sends the sessionID through the
websocket connection. The service can send the client the following messages:

- loggedIn: sessionID is a currently active session
- loggedOut: are we using this??
- revoked: sessionID has been revoked
- register: login process has begun and needs to be redirected to registration
