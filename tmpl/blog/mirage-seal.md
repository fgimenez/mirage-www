# Unikernels: HTTP -> HTTPS

Building a static website is one of the better-supported user stories for Mirage, but it currently results in an HTTP-only site, with no capability for TLS.  Although there's been a great TLS stack [available for a while now](http://openmirage.org/blog/introducing-ocaml-tls), it was a bit fiddly to assemble the pieces of TLS, Cohttp, and the MirageOS frontend tool to result in an HTTPS unikernel until very recently.  With MirageOS 2.5.0, that's changed!  Let's celebrate by building an HTTPS-serving unikernel of our very own.

## Get a Certificate

To serve HTTPS, we'll need a certificate to present to clients (i.e., browsers) for authentication and establishing asymmetric encryption. To just testing things out, or when it's okay to cause a big scary warning message to appear for anyone browsing a site, we can use a self-signed certificate.  Otherwise, the domain name registrar or hosting provider for a site will be happy to sell (or in some cases, give!) a certificate -- both options are explained in more detail below.

### Self-Signed

It's not strictly necessary to get someone else to sign a certificate. We can create and sign our own certificates with the `openssl` command-line tool.  The following invocation will create a secret key in `secrets/server.key` and a public certificate for the domain `totallyradhttpsunikernel.xyz` in `secrets/server.pem`.  (The certificate will be valid for 365 days, so it's a good idea set a calendar reminder to renew it if the service will be up for longer than that.)  The key generated will be a 2048-bit RSA key.

```
openssl req -x509 -newkey rsa:2048 -nodes -keyout secrets/server.key -out secrets/server.pem -days 365 -subj '/CN=totallyradhttpsunikernel.xyz'
```

This key and certificate can be used with `mirage-seal`!  See "Packaging Up a Site with Mirage-Seal" below.

### Signed by Someone Else

Although there are many entities that can sign a certificate with different processes, most have the following in common:

1) you generate a request to have a certificate made for a domain
2) the signing entity requests that you prove your ownership over that domain
3) once verified, the signing entity generates a certificate for you

#### Generating a Certificate-Signing Request

No matter whom we ask to sign a certificate, we'll need to generate a certificate signing request so the signer knows what to create.  The `openssl` command-line tool can do this.  The line below will generate a CSR (saved as server.csr) signed with a 2048-bit RSA key (which will be saved as myserver.key) and hashed with SHA-256, after asking you questions about the domain for which the certificate should be generated.

```
openssl req -nodes -newkey rsa:2048 -sha256 -keyout server.key -out server.csr
```

Once `openssl` has answers to all of its questions, it will generate a `server.csr` that contains the certificate signing request for submission elsewhere.

##### Example: Gandi.net

My domain is registered through the popular registrar Gandi.net.  My registrar did not pay a promotional consideration for this mention, but they do give a free TLS certificate for one year with domain registration, so I elected to have them sign a certificate for me.  Most of this process is managed through their web GUI and a fairly large chunk is automatically handled behind the scenes.  Here's how you can do it too:

Log in to the web interface available through the registrar's website.  You can start the certificate signing process from the "services" tab, which exposes an "SSL" subtab.  Click that (Gandi doesn't need to know that we intend only to support TLS, not SSL).  Hit the "Get an SSL Certificate" button.  Standard SSL is fine.  Even if you're entitled to a free certificate, it will appear that you need to pay here; however at checkout, the total amount due will be 0 in your preferred currency.  Ask for a single address and, if you want to pay nothing, a valid period of 1 year.

Copy the content of the certificate-signing request you generated earlier and paste it into the web form.  Gandi will also ask you to identify your TLS stack; unfortunately `ocaml-tls` isn't in the drop-down menu, so choose OTHER (and perhaps send them a nice note asking them to add the hottest TLS stack on the block to their list).  Click "submit" and click through the order form.

From here on out, the process is automatic if you're buying a certificate for a domain you have registered through them, via the account with which you requested the certificate.  You should shortly receive an e-mail with a subject like "Procedure for the validation of your Standard SSL certificate", which explains the process in more detail, but really all you need to do is wait a while (about 30 minutes, for me).  After the certificate has been generated, Gandi will notify you by e-mail, and you can download your certificate from the SSL management screen.  Click the magnifying glass next to the name of the domain for which you generated the cert to do so.

## Packaging Up a Site with Mirage-Seal

Equipped with a private key and a certificate, let's make an HTTPS unikernel!  First, use `opam` to install `mirage-seal`.  (If `opam` or other MirageOS tooling aren't set up yet, check out the [instructions for getting started](http://openmirage.org/wiki/install).)

```
opam install mirage-seal
```

`mirage-seal` has a few required arguments.  (For more thorough documentation including network configuration settings, see `mirage-seal --help` or [mirage-seal's README file](https://github.com/mirage/mirage-seal/blob/master/README.md).)

* `--data`: one directory containing all the content that should be served by the unikernel.  Candidates for such a directory are the top-level output directory of a static site generator (such as `public` for octopress), the `DocumentRoot` of an Apache configuration, or the `root` of an nginx configuration.
* `--keys`: one directory containing the certificate (`server.pem`) and key (`server.key`) for the site.

To build a Xen unikernel, select the Xen mode with `-t xen` as well.  So in full:

```
mirage-seal --data=/home/me/coolwebsite/public --keys=/home/me/coolwebsite/secrets -t xen
```

`mirage-seal` will then generate a unikernel `mir-seal.xen` and a Xen configuration file `seal.xl` in the current working directory.  To boot it and open the console, invoke `xl create` on the configuration file with the `-c` option:

```
sudo xl create seal.xl -c
```

Via the console, we can see the sealed unikernel boot and obtain an IP through DHCP.  Congratulations -- you made a static site browsable via HTTPS in a unikernel!