# Registration

To register with a certificate authority, a user installs the authenticator
software on a device. After installation, the authenticator should:

- prompt the user for a name for this device
- create an authenticator public/private key pair, ideally stored in hardware
- prompt the user for a username and master password

The authenticator should give the user basic advice about creating a strong
master password, such as ensuring it is at least 15 characters long and random.
They can be advised to write the master password down and keep it in a safe
place or store it in a password manager.

The authenticator then registers with the certificate authority.

## Password Authentication

For ease of development, the authenticator may use standard password
authentication. In this case, the authenticator uses the register endpoint:

### authenticator -> CA

| Method | Path                     | Arguments               |
| ------ | ------------------------ | ----------------------- |
| POST   | /la0.2/password/register | username, password, CSR |

where

```
CSR = certificateSigningRequest(username,authenticatorPublicKey))
```

The potential responses are:

- 200 OK : success, with the body containing authenticatorCertificate
- 403 Forbidden: username already exists

The `authenticatorCertificate` contains (username, authenticatorPublicKey) and
is signed by the `caPrivateKey`.

## PAKE Authentication

For deployment, an authenticator must use a PAKE protocol. PAKE is typically
used to derive a shared key between a client and server, based on the client's
knowledge of a password that is not revealed to the server. In this case, the
authenticator uses PAKE to retrieve an encrypted private key, and then uses this
private key to authorize subsequent transactions with the server, and does not
need a separately-negotiated key.

See [Pake](./pake.md) for details.

## TLS PAKE Authentication

The OPAQUE protocol is being integrated into TLS. When this is available, an
authenticator may use TLS with OPAQUE to negotiate a mutually-authenticated TLS
connection, and then use this connection to authorize subsequent transactions.
