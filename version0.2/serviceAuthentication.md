# Service Authentication

Service authentication begins with a client requesting either registration or
login with a web service.

## Client Request

To authenticate to a web service, the client requests a session object from the
web server:

### client -> web service

| Method | Path               | Arguments |
| ------ | ------------------ | --------- |
| GET    | /la0.2/api/session |           |

The potential responses are:

- 200 OK: success, with the body containing a sessionObject
- 404 Not Found: protocol not supported

A sessionObject consists of:

- domain: the domain of the web service
- sessionID: a unique session ID for this client
- type: the string 'registration' or 'login'

The sesssionObject is signed by the private key of the web service.

## Authenticator Transfer

The client device relays the sessionObject to the authenticator via:

- presenting it as a QR code for scanning by a smartphone authenticator
- providing the information in hidden form fields for a browser extension
  authenticator
- providing the information in a deep link for a desktop authenticator
- transferring it to the authenticator using CTAP2

The authenticator contacts the web service at the given `domain` to get
additional information from the web service:

### authenticator -> web service

| Method | Path            | Arguments |
| ------ | --------------- | --------- |
| GET    | /la0.2/api/info |           |

The potential responses are:

- 200 OK: success, with the body containing the web service's webCertificate
- 404 Not Found: protocol not supported

The webCertificate is used to verify the signature on the sessionObject.

## Service Certificate

Once the authenticator validates the signed object it checks whether it has a
valid _service certificate_ for the desired service. If it does not, it obtains
a service certificate from the CA. To do this, it creates a new service key
pair, `servicePrivateKey` and `servicePublicKey`. If the `type` from the
`sessionObject` is registration, then it generates a random identifier,
`accountID`. Otherwise, it reuses the existing `accountID` for this web service.
The authenticator then generates a CSR:

```
CSR = certificateSigningRequest(accountID,servicePublicKey)
```

that is signed by the `authenticatorPrivateKey`.

### authenticator -> CA

| Method | Path                          | Arguments                                    |
| ------ | ----------------------------- | -------------------------------------------- |
| POST   | /la0.2/user/:username/service | CSR, authSignature, authenticatorCertificate |

where

- authSignature is a signature of the CSR with the `authenticatorPrivateKey`, so
  that the CA can verify the authenticator is authorized to claim ownership of
  the `accountID`.

The potential responses are:

- 200 OK : success, with the body containing serviceCertificate
- 403 Forbidden: accountID claimed by another user

The `serviceCertificate` contains (accountID, servicePublicKey) and is signed by
the `caPrivateKey`.

If the authenticator receives a 403 error upon registration, then it chooses a
new random `accountID`. If the authenticator receives a 403 error upon login,
then their account has been hijacked and an error is reported to the user.

## Session Certificate

Once the authenticator has obtained a valid service certificate, it can
authenticate with the service. If the authenticator is completing registration,
it creates a random secret, `serviceSecret` that it uses for this service only.
Otherwise it finds the `serviceSecret` it has previously used for this web
service.

The authenticator then creates a `sessionCertificate`, which contains
(sessionID, sessionPublicKey) and is signed by the `servicePrivateKey`.

If the authenticator is registering a new account with the web service, then it
sends a POST request to the web service to register the account and authorize
the login:

### authenticator -> service

| Method | Path                | Arguments                               |
| ------ | ------------------- | --------------------------------------- |
| POST   | /la0.2/api/register | serviceCertificate, sessionCertificate, |

The potential responses are:

- 200 OK : success
- 403 Forbidden: invalid certificates

The web service validates the `serviceCertificate` using the `CAPublicKey`. It
validates the `sessionCertificate` using the `servicePublicKey` and ensuring the
`sessionID` is one it recently issued. If these are both valid, the web service
uses the `servicePublicKey` as an account identifier and stores the
`serviceSecret` with that account.

## Registration

If the authenticator is registering a new account with the web service, then it
sends a POST request to the web service to register the account and authorize
the login:

### authenticator -> service

| Method | Path                | Arguments                               |
| ------ | ------------------- | --------------------------------------- |
| POST   | /la0.1/api/register | serviceCertificate, sessionCertificate, |

The potential responses are:

- 200 OK : success
- 403 Forbidden: invalid certificates

The web service validates the `serviceCertificate` using the `CAPublicKey`. It
validates the `sessionCertificate` using the `servicePublicKey` and ensures the
`sessionID` is one it recently issued. If these are both valid, the web service
uses the `servicePublicKey` as an account identifier and stores the
`serviceSecret` with that account.

## Login

If the authenticator is logging into an existing account with the web service,
then it sends a POST request to the web service to authorize the login:

### authenticator -> service

| Method | Path             | Arguments                               |
| ------ | ---------------- | --------------------------------------- |
| POST   | /la0.2/api/login | serviceCertificate, sessionCertificate, |

The potential responses are:

- 200 OK : success
- 403 Forbidden: invalid certificates

The web service validates the `serviceCertificate` using the `CAPublicKey`. It
validates the `sessionCertificate` using the `servicePublicKey` and ensures the
`sessionID` is one it recently issued. It compares the the `serviceSecret` in
the `sessionCertificate` with the one stored in the account. if all of these
checks pass, the web service OKs the login.

## Notifying the client

For both registration and login, the client needs to be notified when
registration or login is complete. With a typical web application, the client
makes a registration or login request directly to the back end, and the response
indicates whether the request was successful. However, for Let's Authenticate,
the authenticator communicates directly with the back end to authorize
registration or login. This decoupling provides some benefits, but also requires
the client to be notified when the request is finished.

The preferred method is for the client to use
[long polling](https://nexocode.com/blog/posts/websockets-friend-or-foe/#:~:text=It%20is%20worth%20to%20mention,by%20a%20reliable%20messaging%20system).
In this case, the client sends a request to the server to check whether the
registration or login is done, and the server doesn't respond until the
authentication is complete. The client can time out the connection if it takes
too long (10s) and try again.

### client -> service

| Method | Path                | Arguments |
| ------ | ------------------- | --------- |
| GET    | /la0.2/api/register | N/A       |
| GET    | /la0.2/api/login    | N/A       |

The potential responses are:

- 200 OK : success
- 403 Forbidden: request denied

Alternatively, the client can open a web socket to the back end, and then the
back end can push a message when the registration or login is done. Long polling
is preferred due to its simplicity -- the back end does not need to support an
extra web socket for these requests.
