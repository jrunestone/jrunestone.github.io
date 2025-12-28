+++
title = "How to use self-signed certificates for development environments in NixOS and Firefox"
template = "post.html"
date = 2025-05-25
path = "/nixos-firefox-self-signed-certs"
[taxonomies]
tags = ["NixOS", "Certificates"]
+++

Use a free SSL certificate with a custom certificate authority (CA) to avoid getting warnings from the browser when using https on localhost.

<!-- toc -->

## Generate a root certificate with `mkcert`
We'll be using [mkcert](https://github.com/FiloSottile/mkcert) to generate our certificates.
Open up a shell with `mkcert` and try to generate and install a root certificate automatically:

```bash
$ nix-shell -p mkcert
$ mkcert -install

Created a new local CA 💥
Installing to the system store is not yet supported on this Linux 😣 but Firefox and/or Chrome/Chromium will still work.
You can also manually install the root certificate at "/home/jr/.local/share/mkcert/rootCA.pem".
Note: Firefox and/or Chrome/Chromium support is not available on your platform. ℹ️
```

## Install the root certificate in NixOS
The certificate was generated for us but we need to install it ourselves.

Reference the path to the root certificate (you can run `mkcert -CAROOT` to see where it is) in your NixOS configuration (do NOT commit the certificate file to a git repository):

```nix
{
  security.pki.certificateFiles = [/path/to/rootCA.pem];
}
```

Rebuild your NixOS configuration.

## Verify the root cert is trusted in Firefox
Open Firefox and open up settings and search for "certificates" hit "View certificates".
Go to the "Authorities" tab and there you should see one ore more entries under "mkcert development CA":

<img src="/images/firefox-certs.png" alt="Firefox certificates view">

## Use custom certs in a web app (ASP.NET Core application example)
First generate a certificate pair with `mkcert`. This certificate will be signed with your custom CA.

```bash
$ mkcert -pkcs12 -p12-file localhost.pfx localhost 127.0.0.1
```

Put the file somewhere safe and then use the following environment variables (or in `appsettings.local.json`) to reference them in your project (the password is hard-coded by `mkcert`):

```bash
ASPNETCORE_Kestrel__Certificates__Default__Path="/path/to/localhost.pfx"
ASPNETCORE_Kestrel__Certificates__Default__Password="changeit"
```
Visit your site - you should get no warnings from the browser:

<img src="/images/firefox-https.png" alt="Firefox secure connection">
