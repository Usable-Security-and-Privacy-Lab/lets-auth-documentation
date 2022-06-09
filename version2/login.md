# Login

The user uses the login functionality to renew the authenticator certificate or
to authorize a new authenticator (either a secondary device or to replace a
primary device).

## Password Authentication

For ease of development, the authenticator may use standard password
authentication. In this case, the authenticator uses the login endpoint:

### authenticator -> CA

| Method | Path                  | Arguments               |
| ------ | --------------------- | ----------------------- |
| POST   | /la0.2/password/login | username, password, CSR |

where

```
CSR = certificateSigningRequest(username,authenticatorPublicKey))
```

The potential responses are:

- 200 OK : success, with the body containing authenticatorCertificate
- 403 Forbidden: incorrect username/password combination

## Pake Authentication

See [Pake](./pake.md) for details.
