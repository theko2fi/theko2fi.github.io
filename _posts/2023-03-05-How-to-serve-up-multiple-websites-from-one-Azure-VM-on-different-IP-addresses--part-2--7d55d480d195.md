---
layout: post
title: "How to serve up multiple websites from one Azure VM on different IP addresses (part 2)"
author: Kenneth KOFFI
categories: [azure]
image: https://images.unsplash.com/photo-1485761954900-f9a29f318567?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=870&q=80
tags: [azure, ssl, apache, httpd, "let's encrypt"]
---

In a previous article [here]({% post_url 2023-02-26-How-to-serve-up-multiple-websites-from-one-Azure-VM-on-different-IP-addresses-740dd6521585-part1 %}), we've seen how to host multiple websites on the same Azure VM with different IP addresses.

We are pleased that we were able to get everything to work. Yet, satisfaction has not yet reached its pinnacle. We currently use the HTTP protocol to access our websites, which is by default not very secure. The HTTPS protocol must be used if we want to improve the security posture of our websites. Getting an SSL certificate is what those who say HTTPS mean.

So I'm going to show you in this part 2, how to obtain Let's Encrypt SSL certificates in order to secure your websites. Millions of websites receive TLS certificates from [Let's Encrypt](https://letsencrypt.org/), a nonprofit Certificate Authority (CA).

## Requirements

To follow along, you must have read the part 1:
  
  [**How to serve up multiple websites from one Azure VM on different IP addresses (part 1)**]({% post_url 2023-02-26-How-to-serve-up-multiple-websites-from-one-Azure-VM-on-different-IP-addresses-740dd6521585-part1 %})

We will continue with our two domain name examples (**foo.com** and **bar.com**) as the previous article.

There are the steps to follow.

## Access to your server terminal

Access to the server terminal via ssh and switch to **root** user.

```bash
sudo -i
```

[Rocky Linux 9](https://rockylinux.org/news/rocky-linux-9-0-ga-release/) is the Operating System (OS) of my Virtual Machine, so you may need to adapt the commands used below according to the Linux distribution you use.

## Install the module mod_ssl

This module provides SSL and TLS support for the Apache HTTP Server.

```bash
dnf install mod_ssl -y
```

## Install **acme.sh**  client

`acme.sh` is an ACME (Automatic Certificate Management Environment) protocol client written in Shell (Unix shell) language. It's probably the **easiest** and **smartest** shell script to automatically issue & renew the free certificates.

Replace `my@example.com` with your email address to be always informed in time if an SSL certificate threatened to expire.

```bash
curl https://get.acme.sh | sh -s email=my@example.com
```

> Note: **After the installation, you must close the current terminal and reopen it to make the change take effect. Don't forget to switch again to root user.**

## Change the CA to Let's Encrypt

`acme.sh` uses [ZeroSSL](https://github.com/acmesh-official/acme.sh/wiki/ZeroSSL.com-CA) as the default Certification Authority. But I want my certs to be issued by Let's Encrypt (it's just a personal preference). So I change the default CA with the command below.

```bash
acme.sh --set-default-ca --server letsencrypt
```

## Issue the certificates

for **foo.com**

```bash
acme.sh --issue -d foo.com -d www.foo.com --webroot /var/www/foo.com/html
```

for **bar.com**

```bash
acme.sh --issue -d bar.com -d www.bar.com --webroot /var/www/bar.com/html
```

* `--issue` the parameter to issue the certificate
* `-d` the parameter to specify the main domain you want to issue the cert for. You must have at least one domain there
* `--webroot` the parameter to specify the web root folder where we host our website files. You **MUST** have write access to this folder.

You should get an output similar to below
![](https://cdn-images-1.medium.com/max/800/1*SQnZlOQxLjNTe6JjzrR4gQ.png)

The certs will be placed in `~/.acme.sh/bar.com/`and `~/.acme.sh/foo.com/`. The certs will be renewed automatically every **60** days.

## Create folders to house the certificate files

for **foo.com**

```bash
mkdir -p /etc/httpd/ssl/foo.com/
```

for **bar.com**

```bash
mkdir -p /etc/httpd/ssl/bar.com/
```

## Install the certificates

After the cert is generated, we **MUST** use this command to install/copy the certs/keys to the expected Apache path.

for **foo.com**

```bash
acme.sh --install-cert -d www.foo.com -d foo.com --cert-file /etc/httpd/ssl/foo.com/cert.pem --key-file /etc/httpd/ssl/foo.com/key.pem --fullchain-file /etc/httpd/ssl/foo.com/fullchain.pem --ca-file /etc/httpd/ssl/foo.com/ca.pem --reloadcmd "service httpd force-reload"
```

for **bar.com**

```bash
acme.sh --install-cert -d www.bar.com -d bar.com --cert-file /etc/httpd/ssl/bar.com/cert.pem --key-file /etc/httpd/ssl/bar.com/key.pem --fullchain-file /etc/httpd/ssl/bar.com/fullchain.pem --ca-file /etc/httpd/ssl/bar.com/ca.pem --reloadcmd "service httpd force-reload"
```

The cert will be renewed every **60** days by default (which is configurable). Once the cert is renewed, the Apache service will be reloaded automatically by the command: `service httpd force-reload`.

## Update the Apache configuration file

Edit (nano, vim...) `/etc/httpd/conf.d/foo.com.conf` file
```xml
<VirtualHost 10.1.0.5:80>
        ServerName www.foo.com
        ServerAlias foo.com

        Redirect permanent / https://foo.com 
</VirtualHost>

<VirtualHost 10.1.0.5:443>
        SSLEngine On
        SSLCertificateFile /etc/httpd/ssl/foo.com/cert.pem
        SSLCertificateKeyFile /etc/httpd/ssl/foo.com/key.pem
        SSLCertificateChainFile /etc/httpd/ssl/foo.com/ca.pem

        ServerName www.foo.com
        ServerAlias foo.com
        DocumentRoot /var/www/foo.com/html
        DirectoryIndex index.php index.htm index.html

        CustomLog "/var/log/httpd/foo.com/access.log" combined
        ErrorLog  "/var/log/httpd/foo.com/error.log"

        <Directory /var/www/foo.com/html>
                AllowOverride All
        </Directory>
</VirtualHost>
```

Edit (nano, vim...) `/etc/httpd/conf.d/bar.com.conf` file
```xml
<VirtualHost 10.1.0.4:80>
        ServerName www.bar.com
        ServerAlias bar.com

        Redirect permanent / https://bar.com 
</VirtualHost>

<VirtualHost 10.1.0.4:443>
        SSLEngine On
        SSLCertificateFile /etc/httpd/ssl/bar.com/cert.pem
        SSLCertificateKeyFile /etc/httpd/ssl/bar.com/key.pem
        SSLCertificateChainFile /etc/httpd/ssl/bar.com/ca.pem

        ServerName www.bar.com
        ServerAlias bar.com
        DocumentRoot /var/www/bar.com/html
        DirectoryIndex index.php index.htm index.html

        CustomLog "/var/log/httpd/bar.com/access.log" combined
        ErrorLog  "/var/log/httpd/bar.com/error.log"

        <Directory /var/www/bar.com/html>
                AllowOverride All
        </Directory>
</VirtualHost>
```

I already explained in the previous article, what most of these directives are for. I will just explain the new ones here.

* `Redirect` this directive maps an old URL into a new one by asking the client to refetch the resource at the new location
* `SSLEngine On` this directive toggles the usage of the SSL/TLS Protocol Engine. By default, the SSL/TLS Protocol Engine is disabled for both the main server and all configured virtual hosts.
* `SSLCertificateFile` this directive points to a file with certificate data in PEM format
* `SSLCertificateKeyFile` this directive points to the PEM-encoded private key file for the server
* `SSLCertificateChainFile` This directive sets the optional _all-in-one_ file where you can assemble the certificates of Certification Authorities (CA) which form the certificate chain of the server certificate. Such a file is simply the concatenation of the various PEM-encoded CA Certificate files, usually in certificate chain order.

When Apache starts up, it has to read the various Certificate ([`SSLCertificateFile`](https://httpd.apache.org/docs/2.4/mod/mod_ssl.html#sslcertificatefile)), Private Key ([`SSLCertificateKeyFile`](https://httpd.apache.org/docs/2.4/mod/mod_ssl.html#sslcertificatekeyfile)) files of the SSL-enabled virtual servers.

## Edit of the Network Security Group (NSG)

Open the port 443/TCP in the Azure NSG, as we did in the previous article for the port 80/TCP.
![](https://cdn-images-1.medium.com/max/800/1*BGPU4v7YxAXs4KZgDRYNtQ.png)

## Restart the Apache service

```bash
systemctl restart httpd
```

...

<iframe src="https://giphy.com/embed/nlJgJxq8kn2mL0frKt" width="800" height="400" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/quixyofficial-its-done-it-is-finally-over-nlJgJxq8kn2mL0frKt"></a></p>


Enjoy your websites with TLS encryption !

In this series of articles, we deployed several websites on the same Azure server with different IP addresses in the first part. Then in this second part, we added SSL/TLS encryption to access our website via HTTPS secure protocol. Thanks for reading this far, and see you soon for new articles.
You can reach me on  the following platforms:

- [LinkedIn](https://www.linkedin.com/in/kenneth-koffi-6b1218178/)
- [Medium](https://theko2fi.medium.com)
- [Github](https://github.com/theko2fi)

