# Kodi Webdav with mTLS (MutualTLS)

This is my solution to enabling Kodi to access a Webdav share served through a proxy that requires mTLS verification of the requesting clients identity.  For me this satisfies my security concerns enough to avoid using a VPN.

### Setup

LibreELEC is used to install Kodi on a client device which is intended to be remote from the media server/webdav share.

The media server/webdav share is only exposed to the public internet via a reverse proxy (Traefik) which requires mTLS verification of the requesting clients identity.

Kodi unfortunately does not support mTLS for Webdav natively, so a local-to-Kodi proxy is needed to handle the mTLS verification and forward the request to Traefik.

### My Solution

Caddy is deployed alongside Kodi in the LibreELEC /storage partition.  A seperate systemd service is used to start Caddy on boot.  Caddy is configured to listen on a local port and forward requests to Traefik, which in turn forwards requests to the Webdav share.  Caddy is configured to be able to verify the TLS certificate of the Traefik router endpoint which is served over HTTPS.  Caddy is also configured to supply the client certificate and key to satisfy the mTLS challenge that Traefik requires.

### Why...

Sftpgo - it has a supported container deployment, provides webdav almost of out the box and includes a 'defender' feature which provides a layer of security against brute force attacks.  I found it easy enough to start with although the documentation for containerised deployments leaves a little to be desired.

Traefik - I've used for a long time as a reverse proxy, its well documented and support, runs as a container and supports mTLS verification of clients.

Caddy - is a simple single-binary proxy with a small footprint something like 40mb vs Traefiks 250mb.  And it supports mTLS challenges out of the box and has decent documentation.

LibreELEC - my client device is a Minisforum S100 so Intel x86_64 architecture.  LE supports this out of the box **alomst**, the UFS storage isn't yet supported by LE but is being added into LE's kernel and image.  LE however supports HDR formats out of the box whereas Kodi ontop of Weyland in mainstream distros isn't quite their yet.

### The Process

**NOTE:  The below is just an illustration to aid understanding and not representative to the actual steps implemented by the protocols at play.**

┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐  
│Kodi    ├─1──►│Caddy   ├─2──►│Traefik ├─5──►│Sftpgo  │  
│        │     │(proxy) │◄──3─┤(proxy) │     │(webdav)│  
│        │     │        ├─4──►│        │     │        │  
│        │◄────┤        │◄────┤        │◄──6─┤        │  
└────────┘     └────────┘     └────────┘     └────────┘ 

- 1. Create a Webdav share in Kodi pointing towards port 80 on the localhost and add the Webdav client account credentials you setup in your Webdav server.
- 2. Caddy is configured to accept the connection request from Kodi and forward it onto the public URL router which Traefik has setup.  This will be something like https://mywebdav.mydomain.com.  Caddy is also configured to forward on the HTTP headers Kodi sent which include the Webdav client account credentials.
- 3. Traefik acknowledges the connection attempt, supplies the TLS certificate for https://mywebdav.mydomain.com and requests the client to comply with a mTLS handshake.  Traefik will automatically preserve the HTTP headers Caddy shared if the mTLS challenge is successfully completed.
- 4. Caddy verifies that the TLS certificate it was provided by Traefik is correct and valid.  Once this is done Caddy then supplies the client certificate and associated private key it has been configured with needed to comply with the mTLS handshake.
- 5. Traefik validates the client certificate and key have been signed by the certificate authority the Traefik router was configured with (mTLS now done) and forwards the connection onto the Webdav server.
- 6. The Webdav server validates the client account details it recevied in the HTTP header, as well as the requested Webdav resource/share for that account.  Provided all match then connection is allowed and data is returned all the way back to Kodi.