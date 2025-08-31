# Root Agency TLS Proxy Test Suite

This is a fork of the official [`badssl.com`](https://badssl.com/) test suite. Since `badssl.com` has been archived and is no longer maintained, this fork adds a test for the **Root Agency** CA certificate (RSA-512, MD5).
Some products (notably *ContentWatch Net Nanny* and *Untangle NG Firewall*) have mistakenly trusted this certificate in the past, exposing users to trivial HTTPS man-in-the-middle attacks.

## Why this test?

We found that certain TLS proxies built into content filters trust the dummy **Root Agency** certificate as a valid issuer of server certificates.  
The `(Microsoft) Root Agency` test in this suite helps determine if your device is vulnerable, either because you are running these products, or another one with the same design flaw.

### Affected products

- **Net Nanny**  (https://www.netnanny.com/): a popular parental control application to restrict and filter bad content from webpages visited by children
  Tested versions: 7.2.4.2, 7.2.6.0, 7.2.8.0 (likely earlier versions too).
  Vendor never confirmed a fix. Subscription required for newer versions: help us test!

- **Untangle NG Firewall**  (https://www.untangle.com/untangle-ng-firewall/): a network appliance to control and filter traffic in enterprise networks
  Tested versions: 13.0, 13.2 (vulnerable).
  Version 14.10 is **no longer vulnerable**.

Since both made the same mistake independently, other products may share the flaw.

---

## How to run the test

1. Install Docker. Clone this repository and run:

   ```bash
   sudo make
   ```
(`sudo` required to start the Docker container at the end)

2. The server will listen on port **443**. Options for testing:
   - Run in a VM and forward (all subdomains of) `badssl.com` to the host, then forward the traffic to the VM using a [TCP proxy](https://www.partow.net/programming/tcpproxy/index.html) or a reverse proxy. Use a DNS proxy like [`dnschef`](https://github.com/iphelix/dnschef) to help you with that. Alternatively, hardcode only `root-agency.badssl.com` in the test machine's `hosts` file.
   - Deploy in a cloud VM and point `badssl.com` to its public IP.
   - Run directly on Linux or possibly WSL.

3. Visit: [https://root-agency.badssl.com](https://root-agency.badssl.com)

### Outcomes

- ❌ **No browser warning** → You are vulnerable.
- ⚠️ **Browser warning shown**:
  - *Unknown CA (e.g., net::ERR_CERT_AUTHORITY_INVALID)*
    - If issuer is **Root Agency** → either no proxy ✅, proxy not filtering this connection ⚠️, or proxy does not trust Root Agency ✅
    - If issuer is a **proxy CA** → the proxy rejected Root Agency, passed back its own “invalid” CA (safe) ✅
  - *Different error* → Proxy intercepted and blocked the connection (safe).

**Note:** SNI is required. All modern browsers support it.

**Note 2:** Content filters may only filter connections based on a list of domain names or keywords to appear in a server certificate. Make sure to mimic or tune the list accordingly.

---

## Why is this dangerous?

TLS was designed to prevent interception. Content filters must break TLS and re-issue certificates, but they also need to validate upstream certificates correctly.

Instead of relying solely on the OS trust store, Net Nanny and Untangle included certificates from Windows’ *Intermediate CA store*, which contains non-trusted certificates like Root Agency. Since Root Agency’s private key is publicly retrievable from Microsoft’s `makecert.exe`, an attacker can forge certificates that these proxies will accept.

## Attack scenario

1. Obtain `makecert.exe` from the Windows SDK.
2. Extract the Root Agency private key (bundled in the binary).
3. Target a victim behind Net Nanny (Windows) or Untangle NG Firewall.
4. Intercept TLS traffic and reissue certificates signed by Root Agency.
   → Victim’s browser sees no warning.

## Root Agency?

Microsoft provides the [*makecert.exe*](https://docs.microsoft.com/en-us/windows/desktop/seccrypto/makecert) certificate management tool as part of its Windows SDK.

By default, when creating a new leaf certificate, makecert chains it to a dummy root certificate called **Root Agency**.

While this root certificate is not trusted by the OS or any browser, it is included in the *Intermediate Certification Authorities* store in every copy of Windows. The root certificate is valid since **1996**, meaning it has probably been there since Windows 95 or 98.

It was also generated with then-current standards and simply carries a **512-bit RSA public key**.
The corresponding private key can simply be retrieved from any version of *makecert.exe* in the PVK resource.

## Timeline of the vulnerabilities

- **2015-03-XX:** Vulnerability found in Net Nanny, PoC works
- **2015-08-24:** Contacted support@netnanny.com about a RSA-512 trusted root certificate
- **2015-08-28:** Response from Net Nanny's VP Development:  *The Net Nanny CA store is created by importing from the browsers/stores we detect. That is, Windows/IE, Mozilla, Android, and Apple/Safari stores. Additionally, we ship a list of CA's that matches Mozilla's CA's (since they use their own store). It is kept up to date with every release, however, we do not update the store again until the next release/update. It would be possible to update the store more frequently via a definition update. An enhancement request to do that has been entered into our system.*
- **2016-02-22:** [Paper presented at NDSS](https://xavier2dc.fr/papers/tls-proxy-ndss2016.pdf)
- **2017-07-28:** Retested current version, Net Nanny is still vulnerable
- **2017-08-09:** Vulnerability found in Untangle NG Firewall (PoC by me, tested by Louis Waked)
- **2018-0X-XX:** L. Waked contacted Untangle, automatic reply only
- **2018-06-06:** [Paper presented at AsiaCCS](https://users.encs.concordia.ca/~mmannan/publications/enterprise-interception-asiaccs2018.pdf)
- **2018-11-12:** Contacted CMU CERT about Net Nanny
- **2018-11-26:** CERT *"decided not to handle the case"* as they *"generally do not accept reports of issues that have already been publicly disclosed."*
- **2018-11-28:** Tested Untangle 14.10, no longer vulnerable
- **2018-11-29:** [Issues initially published](https://web.archive.org/web/20190831061827/https://madiba.encs.concordia.ca/~x_decarn/rootagency.html)
- **2025-08-31:** Badssl.com fork with test added

---

## Contributing

- If you run **Net Nanny** or another filter, please try the Root Agency test and report results.
- Contact details: [https://xavier2dc.fr](https://xavier2dc.fr)
