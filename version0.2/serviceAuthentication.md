# Service Authentication

Service authentication begins with a client requesting either registration or
login with a web service.

## Client Request

To authenticate to a web service, the client requests a session object from the
web server and specifies if the user has selected registration by specifiying a 
`isRegistration` boolean:

### client -> web service

| Method | Path               | Arguments      |
| ------ | ------------------ | -------------- |
| GET    | /la0.2/api/session | isRegistration |

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

## Obtaining New

## Service Certificates for Registration

When registering for an account, the authenticator should first check the
recovery data in case an account has already been created. If the authenticator
has an `accountID` for this service, then it alerts the user to determine
whether they want to register for another account or use the existing one.

When a user has multiple accounts with a service, the authenticator should
include functionality to allow the user to associate a human-readable name with
each `accountID` to distinguish them.

The authenticator should obtain service certificates in batches, as described
below.

## Service Certificates for Login

The authenticator first checks the recovery data to determine whether there is
an existing `accountID` for this service and then checks whether it owns a valid
`service certificate` for this accountID.

If there is no existing `accountID` for this service, then the user must
register for an account first before logging in. The authenticator shows an
error message explaining this to the user.

If the authenticator knows the `accountID` and has a valid service certificate,
it proceeds to obtaining a _session certificate_.

If the authenticator knows the `accountID` and has a valid `servicePrivateKey` and `servicePublicKey` 
for this service, but its `service certificate` has expired, it can renew the service certificate as described below. Otherwise, it may obtain a new service certificate as also described below. 

The authenticator should renew its service certificates before they expire.

## Obtaining New Service Certificates

An authenticator should obtain service certificates in bulk because this (a)
speeds up the account registration process when the user registers with a new
service and (b) avoids leaking to the CA that the authenticator has created a
new account with a service.

To obtain service certificates in a batch, the authenticator creates a
`servicePrivateKey` and `servicePublicKey` for each certificate.

The authenticator then generates a random `accountID` and a CSR for each
`servicePublicKey`:

```
CSR = certificateSigningRequest(accountID,servicePublicKey)
```

Each CSR is signed by the `authenticatorPrivateKey`. The authenticator sends the
CSRs to the authenticator as follows.

### authenticator -> CA

| Method | Path                               | Arguments                                      |
| ------ | ---------------------------------- | ---------------------------------------------- |
| POST   | /la0.2/user/:username/servicebatch | [CSR, authSignature, authenticatorCertificate] |

where

- the argument is a list
- authSignature is a signature of the CSR with the `authenticatorPrivateKey`, so
  that the CA can verify the authenticator is authorized to claim ownership of
  the `accountID`.

The potential responses are:

- 200 OK : success, with the body containing serviceCertificates
- 403 Forbidden: accountID claimed by another user, failed to verify RSA signature for CSR request

If any `accountID` is already taken by another user, the CSR does not return a
`serviceCertificate` for that ID. In the highly unusual case where all
`accountID`s collide, the returned value may be an empty list.

Each `serviceCertificate` contains `(accountID, servicePublicKey)` and is signed
by the `caPrivateKey`.

## Renewing a Service Certificate

When renewing a certificate, the authenticator should have a `servicePrivateKey`
and `servicePublicKey` for this service. When obtaining a new service
certificate, it should create the pair.

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
- 403 Forbidden: accountID claimed by another user, failed to verify RSA signature for CSR request

The `serviceCertificate` contains (accountID, servicePublicKey) and is signed by
the `caPrivateKey`.

If the authenticator receives a 403 error then their account has been hijacked
and the authenticator reports an error to the user.

## Session Certificates

Once the authenticator has obtained a valid service certificate, it can
authenticate with the service. If the authenticator is completing registration,
it creates a random secret, `serviceSecret` that it uses for this service only.
Otherwise it finds the `serviceSecret` it has previously used for this web
service.

The authenticator then creates a `sessionCertificate`, which contains
(sessionID, sessionPublicKey) and is signed by the `servicePrivateKey`.

## Registration

If the authenticator is registering a new account with the web service, then it
sends a POST request to the web service to register the account and authorize
the login:

### authenticator -> web service

| Method | Path                | Arguments                               |
| ------ | ------------------- | --------------------------------------- |
| POST   | /la0.2/api/register | serviceCertificate, sessionCertificate, |

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

### authenticator -> web service

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

### client -> web service

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
