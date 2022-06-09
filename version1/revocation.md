# Revocation

Revocation is available in this version only for authenticators. Any
authenticator can revoke any other authenticator (including itself). This means
it cannot be used for site registration or login until the authenticator logs in
again using the master password. The browser extension and desktop app also
include the option to re-authorize any revoked authenticators.

## Revoke Authenticator

On the page listing all authenticators, the user may select "deauthorize" for
any authenticator. User is prompted for username and master password.

### authenticator -> CA

| Method | Path                 | Arguments                      |
| ------ | -------------------- | ------------------------------ |
| POST   | /users/device/revoke | username, password, deviceCert |

`deviceCert` is the certificate for the authenticator that will be revoked, not
necessarily for the authenticator calling the endpoint

The potential responses are:

- 200 OK: success
- 400 Bad Request: username, password, or deviceCert are bad, or CA otherwise
failed to revoke the deviceCert

## Restore Authenticator

On the authenticators with the "authorize" option for revoked authenticators,
user can restore an authenticator like so:

### authenticator -> CA

| Method | Path                  | Arguments                      |
| ------ | --------------------- | ------------------------------ |
| POST   | /users/device/restore | username, password, deviceCert |

`deviceCert` is the certificate for the authenticator that will be restored, not
necessarily for the authenticator calling the endpoint

The potential responses are:

- 200 OK: success
- so far there is no response to indicate an error occurred, although an error is definitely possible
