# Revocation

An authenticator can revoke both session certificates and authenticator
certificates.

## Session Certificates

An authenticator can revoke a session certificate, which forces logout for that
session. This might be needed if an authenticator is stolen. Alternatively, it
might be needed if the authenticator was used to authorize a session on a client
device that is no longer nearby and needs to be logged out.

### authenticator -> relying party

| Method | Path              | Arguments                              |
| ------ | ----------------- | -------------------------------------- |
| POST   | /la0.2/api/logout | accountCertificate, sessionCertificate |

The potential responses are:

- 200 OK : success
- 403 Forbidden: invalid certificates

If the request is successful, the `sessionCertificate` is considered revoked. No
subsequent requests with that `sessionCertificate` will be honored.

## Authenticator Certificates

A user may need to revoke an authenticator certificate if it is lost or stolen.

### authenticator -> CA

| Method | Path                                  | Arguments                          |
| ------ | ------------------------------------- | ---------------------------------- |
| POST   | /la0.2/user/:username/revoke/password | password, authenticatorCertificate |

The potential responses are:

- 200 OK : success
- 403 Forbidden: invalid certificate

If the request is successful, the `authenticatorCertificate` is considered
revoked. No subsequent requests with that `authenticatorCertificate` will be
honored.

TBD: revoking with PAKE
