---
title: Windows/Git Bash/Curl/Corporate Certificate
excerpt: How to configure Git Bash CURL command to use corporate certificat without admin
---

Recently I had the need to use Windows... where I had to use Git. Thankfully there is a product called [Git Bash](https://gitforwindows.org) which saved me.

In the meantime I had suddenly the need to make some curl requests to HTTPS endpoints AND I'm on a machine which has a corporate security proxy.

I saw errors like:

```bash
foo@bar MINGW64 ~
$ curl https://cdn.azul.com/zulu/bin/zulu11.43.55-ca-jdk11.0.9.1-win_x64.zip --output zulu11.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

In order to get through the proxy a corporate certificate is required.

Now, at that moment I could not invoke admin rights to place the certificate into the mingw install directory as instructed, so how to proceed?

You'll notice that later in the documentation it describes that curl has a environment variable `CURL_CA_BUNDLE` that does not require admin rights modification of the file system.

Edit your `~/.bashrc` file and add the following:

```bash
export CURL_CA_BUNDLE="/c/foo/corp-cert.crt"
```

With that curl commands should start the work.