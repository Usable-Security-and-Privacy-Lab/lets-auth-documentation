# Login

The user uses the login functionality to renew the authenticator certificate or
to authorize a new authenticator to an already existing account.

## Password Authentication

The authenticator uses standard password authentication with the login endpoint:

### authenticator -> CA

| Method | Path                | Arguments                                 |
| ------ | ------------------- | ----------------------------------------- |
| POST   | /authenticate/login | username, password, deviceCert, goodUntil |

where

```
deviceCert = certificateSigningRequest(username, email, authenticatorPublicKey))
```
The potential responses are:

- 200 OK : success, with the body containing `deviceCert`
- 400 Bad Request: denied for invalid username/password or if CA was unable to sign csr `deviceCert`
- 500 Internal Server Error: CA unable to save newly signed `deviceCert`

`deviceCert` contains (username, authenticatorPublicKey) and is signed by the caPrivateKey.
