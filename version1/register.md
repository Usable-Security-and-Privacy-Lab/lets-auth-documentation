# Registration

To register with a certificate authority, a user installs the authenticator
software on a device. After installation, the authenticator:

- prompts the user for a name for this device
- creates an authenticator public/private key pair, stored in available secure hardware
- prompts the user for a username, email, and master password

The authenticator requires a password of at least 8 characters, a unique username,
and a unique device name for each device on their account.

The authenticator then registers with the certificate authority.

Upon registering a browser extension or desktop application authenticator, a
pdf file is provided for download with a list of 10 recovery passkeys. The smartphone
app does not use this because it uses biometrics as alternative authentication
for account recovery.

## Password Authentication

The authenticator uses standard password authentication with the register endpoint:

### authenticator -> CA

| Method | Path                   | Arguments                                        |
| ------ | ---------------------- | ------------------------------------------------ |
| POST   | /authenticate/register | username, password, email, deviceCert, goodUntil |

where

```
deviceCert = certificateSigningRequest(username, email, authenticatorPublicKey))
```
The potential responses are:

- 200 OK : success, with the body containing `deviceCert`
- 400 Bad Request: denied for duplicate username/email or if CA was unable to sign csr `deviceCert`
- 500 Internal Server Error: CA unable to save newly signed `deviceCert`

`deviceCert` contains (username, authenticatorPublicKey) and is signed by the caPrivateKey.
