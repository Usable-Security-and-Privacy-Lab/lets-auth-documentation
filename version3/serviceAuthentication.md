# Service Authentication

Service authentication begins with a client requesting either registration or
login with a relying party.

## Client Request

To authenticate to a relying party, the client requests a session object from
the web server:

### client -> relying party

| Method | Path               | Arguments |
| ------ | ------------------ | --------- |
| GET    | /la3/session/:type |           |

The type can be:

- register
- login

The potential responses are:

- 200 OK: success
- 404 Not Found: protocol not supported

A successful request returns a JSON object with:

- session Object
- signature of the session object

A sessionObject consists of:

- domain: the domain of the relying party
- sessionID: a unique session ID for this client
- type: the string 'registration' or 'login'

The relying party converts the sessionObject to a JSON string and signs it with
its private key. This can be done with the `node:crypto` library, as follows:

```javascript
const sign = createSign("SHA256");
sign.write(sessionString);
sign.end();
const signature = sign.sign(privateKey, "hex");
```

## Authenticator Transfer

The client device relays the sessionObject and the signature to the
authenticator via:

- presenting it as a QR code for scanning by a smartphone authenticator
- providing the information in hidden form fields for a browser extension
  authenticator
- providing the information in a deep link for a desktop authenticator
- transferring it to the authenticator using CTAP2

Once the authenticator obtains this, it contacts the relying party at the given
`domain` to get additional information from the relying party:

### authenticator -> relying party

| Method | Path            | Arguments |
| ------ | --------------- | --------- |
| GET    | /la3/public-key |           |

The potential responses are:

- 200 OK: success, with the body containing the relying party's public key
- 404 Not Found: protocol not supported

The relying party's public key is used to verify the signature on the
sessionObject.

## Account Certificates for Registration

When registering for an account, the authenticator should first check the vault
in case an account has already been created. If the authenticator has an
`accountID` for this relying party, then it alerts the user to determine whether
they want to register for another account or use the existing one.

When a user has multiple accounts with a relying party, the authenticator should
include functionality to allow the user to associate a human-readable name with
each `accountID` to distinguish them.

## Account Certificates for Login

The authenticator first checks the vault to determine whether there is an
existing `accountID` for this relying party and then checks whether it owns a
valid `account certificate` for this accountID.

If there is no existing `accountID` for this relying party, then the user must
register for an account first before logging in. The authenticator shows an
error message explaining this to the user.

If the authenticator knows the `accountID` and has a valid account certificate,
it proceeds to obtaining a _session certificate_.

If the authenticator knows the `accountID` and has a valid `accountPrivateKey`
and `accountPublicKey` for this relying party, but its `account certificate` has
expired, it can renew the account certificate as described below. Otherwise, it
may obtain a new account certificate as also described below.

The authenticator should renew its account certificates before they expire.

## Obtaining or Renewing Account Certificates

To obtain an account certificate, the authenticator needs to generate an
`accountPrivateKey` and `accountPublicKey` for this relying party. When renewing
an account certificate, it should already have this pair for the relying party.

The authenticator then generates a CSR:

```
CSR = certificateSigningRequest(accountID,accountPublicKey)
```

that is signed by the `accountPrivateKey`.

### authenticator -> CA

| Method | Path                        | Arguments                                    |
| ------ | --------------------------- | -------------------------------------------- |
| POST   | /la3/user/:username/account | CSR, authSignature, authenticatorCertificate |

where

- authSignature is an additional signature of the CSR with the
  `authenticatorPrivateKey`, so that the CA can verify the authenticator is
  authorized to claim ownership of the `accountID`.

The potential responses are:

- 200 OK : success
- 403 Forbidden

A successful request contains a JSON object with:

- accountCertificate

The `accountCertificate` contains (accountID, accountPublicKey) and is signed by
the `caPrivateKey`. It has a lifetime of 1 minute.

The CA will perform the following checks before issuing the account certificate:

- an account exists for this username
- the account ID has not already been claimed by another user
- verify the `authenticatorCertificate`
  - it is signed by the CA's private key
  - the username in the certificate matches the username in the API request
- verify the `authSignature` of the CSR using `authenticatorPublicKey` from the
  `authenticatorCertificate`
- verify the signature in the CSR using `accountPublicKey`

If any of these verifications fail, the server returns 403 Forbidden and a text
message in the body indicating the reason for the error.

If an authenticator receives a 403 error when obtaining an account certificate
for an existing accountID (for login, not registration), then their account has
been hijacked and the authenticator reports an error to the user.

## Session Certificates

Once the authenticator has obtained a valid account certificate, it can
authenticate with the relying party. If the authenticator is completing
registration, it generates a new key pair, `sessionPublicKey` and
`sessionPrivateKey` that is unique to this relying party. It must then
synchronize this key pair with the vault. When logging in, the authenticator
should find the key pair it has previously used for this relying party in the
vault.

The authenticator then creates a `sessionCertificate`, which contains
(sessionID, sessionPublicKey) and is signed by the `accountPrivateKey`.

## Registration

If the authenticator is registering a new account with the relying party, then
it sends a POST request to the relying party to register the account and
authorize the login:

### authenticator -> relying party

| Method | Path          | Arguments                              |
| ------ | ------------- | -------------------------------------- |
| POST   | /la3/register | accountCertificate, sessionCertificate |

The potential responses are:

- 200 OK : success
- 403 Forbidden: invalid certificates

The relying party validates the `accountCertificate` using the `CAPublicKey`.
For demo purposes, the replying party can be configured with the `CAPublicKey`.
The relying party validates the `sessionCertificate` using the
`accountPublicKey` and ensures the `sessionID` is one it recently issued. If
these are both valid, the relying party uses the `accountID` as an account
identifier and stores the `sessionPublicKey` with that account.

## Login

If the authenticator is logging into an existing account with the relying party,
then it sends a POST request to the relying party to authorize the login:

### authenticator -> relying party

| Method | Path       | Arguments                              |
| ------ | ---------- | -------------------------------------- |
| POST   | /la3/login | accountCertificate, sessionCertificate |

The potential responses are:

- 200 OK : success
- 403 Forbidden: invalid certificates

The relying party validates the `accountCertificate` using the `CAPublicKey`. It
validates the `sessionCertificate` using the `accountPublicKey` and ensures the
`sessionID` is one it recently issued. It compares the `sessionPublicKey` in the
`sessionCertificate` with the one stored in the account. if all of these checks
pass, the relying party OKs the login.

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

### client -> relying party

| Method | Path                                | Arguments |
| ------ | ----------------------------------- | --------- |
| GET    | /la3/api/register?session=sessionID | N/A       |
| GET    | /la3/api/login?session=sessionID    | N/A       |

The potential responses are:

- 200 OK : success
- 403 Forbidden: request denied

Alternatively, the client can open a web socket to the back end, and then the
back end can push a message when the registration or login is done. Long polling
is preferred due to its simplicity -- the back end does not need to support an
extra web socket for these requests.

## Logout

The client can logout using the following method.

### client -> relying party

| Method | Path                          | Arguments |
| ------ | ----------------------------- | --------- |
| GET    | /la3/logout?session=sessionID | N/A       |

The potential responses are:

- 200 OK : success
- 403 Forbidden: request denied
