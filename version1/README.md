# Version 1

This document describes Version 1 of the Let's Authenticate System.

## Roadmap

The documentation for Let's Authenticate is covered in the following:

- [Registration](./register.md): Describes process of registering user with a
  1Key account on an authenticator.

- [Login](./login.md): Describes process of logging user in onto their 1Key
  account on an authenticator.

- [Service Authentication](./serviceAuthentication.md): Describes how client
  initiates a login request with a site (or phone app) and transfers pertinent
  information to the authenticator. Describes how authenticator communicates
  with the CA to obtain site certificates and register or login a user with a
  site. Explains the client websocket that notifies the client of login status.

- [Account Recovery](./accountRecovery.md): Describes how authenticator
  retrieves and receives updated information on account authenticators and site
  certificates. Describes how an account is recovered if the user forgets the
  master password.

- [Revocation](./revocation.md): Describes how an authenticator interacts with
  the CA to revoke and restore any authenticator.
