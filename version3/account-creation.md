# Account Creation

The authenticator relies on the web browser to create an account with the CA using FIDO2. Once the account is created, the browser can authorize the authenticator for this user's account by obtaining an authenticator certificate.

The authenticator takes the following steps.

## Initiating registration with the browser

The authenticator starts by asking the user for:

- a username
- a password (or authorization to use a biometric if the authenticator is a phone)

> If the authenticator is a browser extension, it uses the password to derive a symmetric key, which it uses to encrypt data it stores locally.

> If the authenticator is a phone, it ... (TBD)

The authenticator then generates n `authenticator key pair`, $K_{auth}, S_{auth}$.

Once this is done, the authenticator opens a new browser tab, using the following URL:

`http://api.letsauth.org/register/{username}/?authPublicKey={authPublicKey}`
            
where

`authPublicKey = $K_{auth}`

This will allow the user to create an account with the CA and simultaneously authorize $K_{auth}$ for their account.

## Browser interactions with the CA

The browser will then use the Let's Authenticate CA web site to register an account with the CA, using the FIDO2 protocol. The browser calls the following endpoint:

### browser → CA

| Method | Path                                  | Body |
| ------ | ------------------------------------- | ---- |
| GET    | /la3/account/create-begin/:username | -    |

Possible return values include:

- 200 OK : success
- 403 Forbidden: username already exists
- 50X: any other errors

If the username already exists, the browser should inform the user that the username already exists so they can retry with a different username.

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
| POST   | /la3/account/create-finish/:username | credential, authPublicKey |

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
- authPublicKey

The CA adds the `authPublicKey` to a list of authorized authenticator public keys for that user's account.

## Getting an authenticator certificate

Once the user finishes creating their account with the CA, the authenticator should prompt them to finish the account creation process.

> If the authenticator is a phone, it should be able to include a link that, when clicked, opens the authenticator app to indicate the user is ready to move to the next step.

To obtain an authenticator certificate, the authenticator generates a
certificate signing request (CSR) that contains only the username and this key
pair and no identifying information. It then calls this endpoint:


### authenticator → CA

| Method | Path                                   | Body                         |
| ------ | -------------------------------------- | ---------------------------- |
| POST   | /la3/account/sign-csr/:username | CSR |

Possible return values include:

- 200 OK : success
- 403 Forbidden: there is no authentication cookie present, the CSR is not properly signed,
  or the username in the CSR is not owned by this user
- 50X: any other errors

The response includes an `authenticator certificate` if no other error occurs.


The CA verifies that the CSR is signed properly and then generates and returns
an `authenticator certificate`. This certificate contains (username, $K_{auth}$)
and is signed by the CA's private key.

If any step of this process fails with a 40X error, then the authenticator
explains to the user why the error occured.

If any step of this process fails with a 50X error, then the authenticator
informs the user that the CA is not available.