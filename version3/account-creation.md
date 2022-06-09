# Account Creation

To create an account with a certificate authority (CA), a user installs the
authenticator software on a device. After installation, the authenticator should

- prompt the user for a name for this device and
- create an account with the CA

## Creating an account

To create an account with the CA, the authenticator uses FIDO2. It starts by
creating an `authenticator key pair`, $K_{auth}, S_{auth}$, and then generates a
certificate signing request (CSR) that contains only the username and this key
pair and no identifying information. The default expiration time for the CSR
should be 10 days.

Once this is done, the authenticator calls the following endpoint:

### authenticator → CA

| Method | Path                                  | Body |
| ------ | ------------------------------------- | ---- |
| GET    | /la0.3/account/create/begin/:username | -    |

Possible return values include:

- 200 OK : success
- 403 Forbidden: username already exists
- 50X: any other errors

If the username already exists, the authenticator should abort the account
creation process and inform the user that the username already exists.

The response to this request contains `credential creation options` that guide
the authenticator on how to use FIDO2 to authenticate with the CA.

**If the authenticator is a browser extension, it sets the relying party ID for
the public key in the credential creation options to be equal to the browser
extension ID.**

The authenticator then interacts with the browser to initiate FIDO2
registration.

**If the authenticator is a browser extension, it calls
`navigator.credentials.create()`, passing it the public key of the CA, from the
credential creation options.**

When the authenticator finishes interacting with the user, it receives a
credential:

- id
- rawId
- type
- response
  - attestationObject
  - clientDataJSON

It uses this credential to call this endpoint:

### authenticator → CA

| Method | Path                                   | Body                         |
| ------ | -------------------------------------- | ---------------------------- |
| POST   | /la0.3/account/create/finish/:username | credential, CSR, backupCodes |

Possible return values include:

- 200 OK : success
- 403 Forbidden: the FIDO2 registration fails, the CSR is not properly signed,
  or the username in the CSR is not owned by this user
- 50X: any other errors

The rawId, attestationObject, and clientDataJSON must all be base-64 encoded.

Note that the combination of credential and CSR are sent using JSON as:

- credential
  - id
  - rawId
  - type
  - response
    - attestationObject
    - clientDataJSON
- CSR

The CA verifies that the CSR is signed properly and then generates and returns
an `authenticator certificate`. This certificate contains (username, $K_{auth}$)
and is signed by the CA's private key.

If any step of this process fails with a 40X error, then the authenticator
explains to the user why the error occured.

If any step of this process fails with a 50X error, then the authenticator
informs the user that the CA is not available.
