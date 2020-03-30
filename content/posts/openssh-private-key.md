+++
title = "Converting openssh private key to PEM format"
date = 2020-03-31
tags = ["openssh"]
draft = false
+++

Newer version of the `openssh` will use their own format to store a private
key:

```conf
-----BEGIN OPENSSH PRIVATE KEY-----
BASE64KEYTEXT
-----END OPENSSH PRIVATE KEY-----
```

The traditional format of the key can sometimes be useful (or even required -
for example [guacamole](https://guacamole.apache.org/) will not be able to use the `OPENSSH` key format).

To convert the key use `ssh-keygen`. The command will overwrite the private
key file, so if the original key should be preserved make sure to create a
backup:

```bash
ssh-keygen -p -m PEM -f ~/.ssh/id_rsa_test
```

<a id="code-snippet--Output"></a>
```conf
-----BEGIN RSA PRIVATE KEY-----
BASE64KEYTEXT
-----END RSA PRIVATE KEY-----
```
