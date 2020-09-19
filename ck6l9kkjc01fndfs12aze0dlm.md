## Cloudflare for CDN, Netlify for hosting

Netlify is great for hosting your static content, but it comes with some limitation. There is bandwidth cap and CDN/security settings are less advanced compared to Cloudflare. When hosting a small, static site for your hobby project or side hustle, it makes a lot of sense to marry those two services for the best outcome possible.

This article represents opinionated view on optimal configuration that results in highest Security level achievable in free tiers on both platforms.

# Cloudflare to Netlify
Delegate your domain to be managed by Cloudflare,not Netlify. This will achieve few things:
- your traffic will flow through Cloudflare reverse proxies. Aparently Cloudflare have more PoPs and will cache your site as well as provide last resort "always on-line" resiliency.
- Cloudflare caching will save your Netlify bandwidth
- DNS will be managed by Cloudflare servers directing traffic to geographically closest PoP.
- Security will be managed by Cloudflare. This allows you to achieve **A+** grade. You will also need HSTS to get this grade.

Easiest way to redirect your site to Netlify is to create CNAME record in your Cloudflare zone file pointing to Netlify *Default subdomain*. Here is an example:
![Annotation 2020-02-13 212910.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1581629383667/qN4wIBAil.png)

Note, that for this to work you must set your *primary domain* in Netlify accordingly. Ignore *Check DNS configuration* warning, Netlify just want to control your domain, but redirecting to them via CNAME record is more than enough.

Further recommendations for Cloudflare:
- Set "Minimum TLS Version" to TLS 1.2. When left at default level, [Qualys](https://blog.qualys.com/ssllabs/2018/11/19/grade-change-for-tls-1-0-and-tls-1-1-protocols) will cap your site score at B. You definitely want **A+**.
- Enable "Automatic HTTPS Rewrites" - this should be paired with HSTS enablement.

# Cloudflare Origin certificates
The above configuration, while providing **A+** rating, has severe drawback. Netlify uses *certbot* to renew Let's Encrypt SSL certificate. As our configuration uses opportunistic HTTP to HTTPS upgrades, *certbot* is unable to access .well-known directory via HTTP protocol, breaking the domain validation process. As such Let's Encrypt certificate cannot be issued/renewed. There are three potential solution for this problem:
- Reduce security by changing *encryption mode* from **Full** to **Flexible**.
- Use Page Rules to allow HTTP protocol against .well-know URLs.
- Provision Netlify site with Cloudflare Origin certificates.Those are valid for 15 years, what makes rotation problem beyond site longevity.

Here is how to implement the last option:
1. On Cloudflare click *generate certificate* under *Origin server* tab. I strongly recommend not to generate apex certificate. It is much more secure to specify exact server name(s). Cloudflare will provide two PEM encoded cryptographic materials: private key and certificate.
2. On Netlify,go into *domain settings / HTTPS* and click *Install custom certificate*. Netlify will ask for certificate, private key and intermediate cert. The first two should be directly copied from Cloudflare.
![Annotation 2020-02-13 212910.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1581638041453/dJdcx9o4S.png)

Netlify does not accept empty value for *intermediate cert*. So please use one of the following Cloudflare Origins ROOTs (technically web servers should not serve them, as roots must be present in client truststore, but this is why we go for Cloudflare security, isn't it?).

Use this for *intermediate cert* if you generated RSA Cloudflare Origin certificate:
```
-----BEGIN CERTIFICATE-----
MIIEADCCAuigAwIBAgIID+rOSdTGfGcwDQYJKoZIhvcNAQELBQAwgYsxCzAJBgNV
BAYTAlVTMRkwFwYDVQQKExBDbG91ZEZsYXJlLCBJbmMuMTQwMgYDVQQLEytDbG91
ZEZsYXJlIE9yaWdpbiBTU0wgQ2VydGlmaWNhdGUgQXV0aG9yaXR5MRYwFAYDVQQH
Ew1TYW4gRnJhbmNpc2NvMRMwEQYDVQQIEwpDYWxpZm9ybmlhMB4XDTE5MDgyMzIx
MDgwMFoXDTI5MDgxNTE3MDAwMFowgYsxCzAJBgNVBAYTAlVTMRkwFwYDVQQKExBD
bG91ZEZsYXJlLCBJbmMuMTQwMgYDVQQLEytDbG91ZEZsYXJlIE9yaWdpbiBTU0wg
Q2VydGlmaWNhdGUgQXV0aG9yaXR5MRYwFAYDVQQHEw1TYW4gRnJhbmNpc2NvMRMw
EQYDVQQIEwpDYWxpZm9ybmlhMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC
AQEAwEiVZ/UoQpHmFsHvk5isBxRehukP8DG9JhFev3WZtG76WoTthvLJFRKFCHXm
V6Z5/66Z4S09mgsUuFwvJzMnE6Ej6yIsYNCb9r9QORa8BdhrkNn6kdTly3mdnykb
OomnwbUfLlExVgNdlP0XoRoeMwbQ4598foiHblO2B/LKuNfJzAMfS7oZe34b+vLB
yrP/1bgCSLdc1AxQc1AC0EsQQhgcyTJNgnG4va1c7ogPlwKyhbDyZ4e59N5lbYPJ
SmXI/cAe3jXj1FBLJZkwnoDKe0v13xeF+nF32smSH0qB7aJX2tBMW4TWtFPmzs5I
lwrFSySWAdwYdgxw180yKU0dvwIDAQABo2YwZDAOBgNVHQ8BAf8EBAMCAQYwEgYD
VR0TAQH/BAgwBgEB/wIBAjAdBgNVHQ4EFgQUJOhTV118NECHqeuU27rhFnj8KaQw
HwYDVR0jBBgwFoAUJOhTV118NECHqeuU27rhFnj8KaQwDQYJKoZIhvcNAQELBQAD
ggEBAHwOf9Ur1l0Ar5vFE6PNrZWrDfQIMyEfdgSKofCdTckbqXNTiXdgbHs+TWoQ
wAB0pfJDAHJDXOTCWRyTeXOseeOi5Btj5CnEuw3P0oXqdqevM1/+uWp0CM35zgZ8
VD4aITxity0djzE6Qnx3Syzz+ZkoBgTnNum7d9A66/V636x4vTeqbZFBr9erJzgz
hhurjcoacvRNhnjtDRM0dPeiCJ50CP3wEYuvUzDHUaowOsnLCjQIkWbR7Ni6KEIk
MOz2U0OBSif3FTkhCgZWQKOOLo1P42jHC3ssUZAtVNXrCk3fw9/E15k8NPkBazZ6
0iykLhH1trywrKRMVw67F44IE8Y=
-----END CERTIFICATE-----
``` 

Use this, if your Cloudflare Origin certificate is EC:
```
-----BEGIN CERTIFICATE-----
MIICiTCCAi6gAwIBAgIUXZP3MWb8MKwBE1Qbawsp1sfA/Y4wCgYIKoZIzj0EAwIw
gY8xCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlhMRYwFAYDVQQHEw1T
YW4gRnJhbmNpc2NvMRkwFwYDVQQKExBDbG91ZEZsYXJlLCBJbmMuMTgwNgYDVQQL
Ey9DbG91ZEZsYXJlIE9yaWdpbiBTU0wgRUNDIENlcnRpZmljYXRlIEF1dGhvcml0
eTAeFw0xOTA4MjMyMTA4MDBaFw0yOTA4MTUxNzAwMDBaMIGPMQswCQYDVQQGEwJV
UzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZyYW5jaXNjbzEZ
MBcGA1UEChMQQ2xvdWRGbGFyZSwgSW5jLjE4MDYGA1UECxMvQ2xvdWRGbGFyZSBP
cmlnaW4gU1NMIEVDQyBDZXJ0aWZpY2F0ZSBBdXRob3JpdHkwWTATBgcqhkjOPQIB
BggqhkjOPQMBBwNCAASR+sGALuaGshnUbcxKry+0LEXZ4NY6JUAtSeA6g87K3jaA
xpIg9G50PokpfWkhbarLfpcZu0UAoYy2su0EhN7wo2YwZDAOBgNVHQ8BAf8EBAMC
AQYwEgYDVR0TAQH/BAgwBgEB/wIBAjAdBgNVHQ4EFgQUhTBdOypw1O3VkmcH/es5
tBoOOKcwHwYDVR0jBBgwFoAUhTBdOypw1O3VkmcH/es5tBoOOKcwCgYIKoZIzj0E
AwIDSQAwRgIhAKilfntP2ILGZjwajktkBtXE1pB4Y/fjAfLkIRUzrI15AiEA5UCL
XYZZ9m2c3fKwIenMMojL1eqydsgqj/wK4p5kagQ=
-----END CERTIFICATE-----
``` 

