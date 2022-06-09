# Account Recovery and Synchronization

Previously elobs/recoveryData were used, but this working version does not have
them in use. All important account information for each user is stored with the
CA, unencrypted(?). Authenticators synchronize with one another using a websocket
connection with the CA, along with other periodic retrieval of information.

## Retrieval

The authenticator retrieves all authenticator and accounts information upon
logging in or registering a new authenticator.
The browser extension also includes a feature for the user to "sync" their
account, retrieving all information in this same way.

### authenticator -> CA

| Method | Path         | Arguments                                                     |
| ------ | ------------ | ------------------------------------------------------------- |
| POST   | /users/certs | username, deviceCert, dateString, signedDataString, lastFetch |

where:

- `deviceCert`: certificate for this authenticator
- `dateString`: the ISO string of the current time
- `signedDataString`: `dateString` signed with the authenticator's private key
- `lastFetch`: an ISO string. This date will determine which information is sent
back. The CA returns a body of all authenticators certs and site certs created
or updated since this time.

The potential responses are:

- 200 OK: success, with the body of user account information
- 400 Bad Request: denied because `signedDataString` was not verified,
invalid/old `dateString`, invalid `lastFetch` date string, or invalid `deviceCert`.

The body of user account information includes a list of all authenticator names
and certificates, each containing a sublist of domains registered with that
authenticator and their corresponding siteCerts.

## Websocket

To allow for live updates of accounts/authenticators, a websocket connects the
authenticator to the CA upon login or registration. The authenticator opens the
websocket with the path /users/cert/websocket.

Whenever a certificate (authenticator or site) is created or updated with the
CA, or an authenticator's revocation status changes, the websocket pushes the
certificate type, certificate, and other pertinent information to all connected
authenticators. Authenticators listen for these messages and update accordingly.

## Forgot Password

This feature is currently functional on the browser extension and desktop app(?)
If the user forgets their password, this allows them to change the master
password with the CA.

User must first provide one of the recovery codes provided at registration.
Recovery codes are stored on the authenticator, NOT the CA and are unique to
each authenticator.
Then user is prompted for username and new master password and the recovery
endpoint is called:

### authenticator -> CA

| Method | Path                   | Arguments                                           |
| ------ | ---------------------- | --------------------------------------------------- |
| POST   | /users/account/recover | username, password, deviceCert, certList, signature |

where

- `password`: new master password provided by user
- `deviceCert`: authenticator certificate for this device
- `certList`: currently unused (previously used as a list of eblobs/recovery data)
- `signature`: currently unused (previously was the certList string signed with
  the authenticator private key)

The potential responses are:

- 204 No Content: success, no response body
- 400 Bad Request: failed to verify certList, currently NA
- 500 Internal Server Error: password failed to change
