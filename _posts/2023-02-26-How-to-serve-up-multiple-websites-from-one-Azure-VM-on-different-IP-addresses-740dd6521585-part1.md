---
layout: post
title: "How to serve up multiple websites from one Azure VM on different IP addresses (part 1)"
author: Kenneth KOFFI
categories: [azure]
image: https://cdn-images-1.medium.com/max/800/1*h_mAXPrdsSlhmN3PdW4nJA.jpeg
tags: [azure, apache, webserver]
---

In this article, I'm going to show you how to host multiple websites on the same Azure server with different IP addresses. This means that we will have a different IP address for each website (domain name). The fact that they are running on the same physical server won't be apparent to the end user.

As it is 2023, I won't go into detail about what a website is or how crucial it is for your project, business, or brand. So let's get right to the tutorial's main point.

## Requirements

To follow this tutorial, all you will need is:

* an Azure subscription
* at least two domain names
* basic knowledge on Azure services and Linux systems

In this article, I'm going to use **foo.com** and **bar.com** as example domain names. Obviously, you'll be using different domains. Replace them by yours.

## Microsoft Azure

Sign in to the [Azure portal](https://azure.microsoft.com/en-us/)

> _Note:_ **The Azure region where you create each of the resources listed below should be the same. I've chosen North Europe as example for this write up.**

### Create a resource group

Let's create a resource group that will house all the resources associated to this project. In this example, I will name mine `websites-RG`. Feel free to use whatever name you want.
![](https://cdn-images-1.medium.com/max/800/1*MkNlZGMUWyIbacUaTPVGJA.png)

### Create the public IP address for each website

Let's create two IP addresses in the `websites-RG` resource group.
![](https://cdn-images-1.medium.com/max/800/1*MZrjH0oic0LdDmlCudaNwg.png)

**foo.com** public IP

![](https://cdn-images-1.medium.com/max/800/1*ySF8L6SC40eaT4JhdQYxmw.png)
![](https://cdn-images-1.medium.com/max/800/1*DvY2rqPc36yXp_X4N6fpiQ.png)

**bar.com** public IP

![](https://cdn-images-1.medium.com/max/800/1*ZuAhpeFSHab3u6xvSzSIow.png)
![](https://cdn-images-1.medium.com/max/800/1*DvY2rqPc36yXp_X4N6fpiQ.png)

### Create the Virtual Machine

Let's create the virtual machine that will host the websites.
![](https://cdn-images-1.medium.com/max/800/1*aJK355sFhuhtsRhBTpD5nw.png)

I decided to create the VM with none public IP. I will attach later the ones we created in the previous section.
![](https://cdn-images-1.medium.com/max/800/1*An2A_7-1SRCw1GJxpLJFtw.png)

### Associate the IP addresses to the Virtual Machine

Go to the networking settings of the website-VM
![](https://cdn-images-1.medium.com/max/800/1*Ja6jPeZLzFzM1riF0L5iEA.png)

Select the Network Interface Card (NIC) of the VM
![](https://cdn-images-1.medium.com/max/800/1*IzSFlZc1cqm8PU7ct0ObiA.png)
![](https://cdn-images-1.medium.com/max/800/1*HwZ4kixxoHfr3JQNokJHuA.png)

**Double-click** on ipconfig1 to edit
![](https://cdn-images-1.medium.com/max/800/1*0i5p9OwgXUgNS3bZ44xayw.png)

Associate one of the public IP created previously and make sure to select the **static** assignment for the **private IP**. I will explain why later.
![](https://cdn-images-1.medium.com/max/800/1*LZhXFjk4KsrVRbRoKvkkzA.png)

Add a second IP configuration to the NIC of the VM for the second website (domain name). Add as many as necessary if you have more websites.
![](https://cdn-images-1.medium.com/max/800/1*q0yYeTCu0OqaDmJyJQZwYQ.png)

Associate the second public IP address created in the previous section. Assign a static private IP address too.
![](https://cdn-images-1.medium.com/max/800/1*963yq8IuVhjxQzmBzdpVwQ.png)

Finally, our NIC's IP configurations look like this
![](https://cdn-images-1.medium.com/max/800/1*0DV0lhZmpmm65iqJIlfDCw.png)

- **bar.com**
    - public IP address: 20.67.188.162
    - private IP address: **10.1.0.4**
- **foo.com**
    - public IP address: 20.67.187.123
    - private IP address: **10.1.0.5**

**As we can see, both private IP addresses are in the same subnet address range of the** `website-vNet` **virtual network**. This is important. Take note of the **private** IP address assigned to each domain name. We will use it later.

### Edit of the Network Security Group (NSG)

Create an inbound security rule on the Network Security Group (NSG) to allow incoming HTTP request to reach the VM on port 80/TCP.
![](https://cdn-images-1.medium.com/max/800/1*HAfbvcm58o0SjiaooExBjw.png)

### Restart the Virtual Machine

Restart the VM for the changes to take effect
![](https://cdn-images-1.medium.com/max/800/1*G8J45dv6BjDytSzabX4B9A.png)

## Server configuration

Access to the server terminal via ssh.

[Rocky Linux 9](https://rockylinux.org/news/rocky-linux-9-0-ga-release/) is the Operating System (OS) of my Virtual Machine, so you may need to adapt the commands used below according to the Linux distribution you use.

### Switch to root user

```bash
sudo -i
```

### Upgrade the system

```bash
dnf upgrade -y
```

### Install Apache

```bash
dnf install httpd -y
```

### Start and enable the Apache service

```bash
systemctl start httpd
systemctl enable httpd
```

### Configure the sites

Create the folder to hold `foo.com` website's files

```bash
mkdir -p /var/www/foo.com/html
```

My website's code is a straightforward **index.html** file that will display "Welcome to <domain name> !!" based on the domain.

Edit (nano, vim...) `/var/www/foo.com/html/index.html` file and paste the following content to it
```html
<html>
  <body>
      <h2>Welcome to foo.com !!</h2>
  </body>
</html>
```

Create the folder to hold the `bar.com` website's files

```bash
mkdir -p /var/www/bar.com/html
```

Edit (nano, vim...) `/var/www/bar.com/html/index.html` file and paste the following content inside
```html
<html>
  <body>
      <h2>Welcome to bar.com !!</h2>
  </body>
</html>
```

Each site needs its own file system directory. You'll place the website's files, such as HTML, CSS and JavaScript, within this directory. It's termed the `DocumentRoot` by Apache, as it's the root from which documents are served.

### Configure Apache

We need to create a Virtual Host file for each of the site. These files define how Apache serves the websites. The term **Virtual Host** refers to the practice of running more than one website on a single machine. Virtual hosts can be "[IP-based](https://httpd.apache.org/docs/2.4/vhosts/ip-based.html)", meaning that you have a different IP address for every website, or "[name-based](https://httpd.apache.org/docs/2.4/vhosts/name-based.html)", meaning that you have multiple names running on each IP address. In our case, we're going to use the **IP-based** Virtual Host.

1. Create the configuration file `/etc/httpd/conf.d/foo.com.conf` for **foo.com** and paste the following content

```xml
<VirtualHost 10.1.0.5:80>
        ServerName www.foo.com
        ServerAlias foo.com
        DocumentRoot /var/www/foo.com/html
        DirectoryIndex index.php index.htm index.html

        CustomLog "/var/log/httpd/foo.com/access.log" combined
        ErrorLog  "/var/log/httpd/foo.com/error.log"

        <Directory /var/www/foo.com/html>
                AllowOverride None
                Order deny,allow
                Deny from all
                Allow from all
        </Directory>
</VirtualHost>
```

2. Create the configuration file `/etc/httpd/conf.d/bar.com.conf` for **bar.com** and paste the following content

```xml
<VirtualHost 10.1.0.4:80>
        ServerName www.bar.com
        ServerAlias bar.com
        DocumentRoot /var/www/bar.com/html
        DirectoryIndex index.php index.htm index.html

        CustomLog "/var/log/httpd/bar.com/access.log" combined
        ErrorLog  "/var/log/httpd/bar.com/error.log"

        <Directory /var/www/bar.com/html>
                AllowOverride None
                Order deny,allow
                Deny from all
                Allow from all
        </Directory>
</VirtualHost>
```

There's the big trap of this mission.

> **Note: The IP address used in the <VirtualHost> directive should be the **private** IP address assigned to each site (domain name) in the IP configurations of the NIC.**

Why ?

Because once inbound internet traffic arrives at the public IP address, Azure translates the public address to the private IP address. This means all the network packets arriving at your VM's NIC have the private IP addresses as destination.

The [VirtualHost](https://httpd.apache.org/docs/2.4/mod/core.html#virtualhost) directive in the configuration file is also used to set the values of [ServerAdmin](https://httpd.apache.org/docs/2.4/mod/core.html#serveradmin), [ServerName](https://httpd.apache.org/docs/2.4/mod/core.html#servername), [DocumentRoot](https://httpd.apache.org/docs/2.4/mod/core.html#documentroot), [ErrorLog](https://httpd.apache.org/docs/2.4/mod/core.html#errorlog) and [TransferLog](https://httpd.apache.org/docs/2.4/mod/mod_log_config.html#transferlog) or [CustomLog](https://httpd.apache.org/docs/2.4/mod/mod_log_config.html#customlog) configuration directives to different values for each virtual host.

* `ServerName` this directive sets the request scheme, hostname and port that the server uses to identify itself.
* `ServerAlias` this directive sets the alternate names for a host
* `DocumentRoot` this directive sets the directory from which [`httpd`](https://httpd.apache.org/docs/2.4/programs/httpd.html) will serve files.
* `DirectoryIndex` this directive sets the list of resources to look for, when the client requests an index of the directory by specifying a / at the end of the directory name.
* `ErrorLog` this directive sets the name of the file to which the server will log any errors it encounters
* `<Directory>` and `</Directory>` are used to enclose a group of directives that will apply only to the named directory, subdirectories of that directory, and the files within the respective directories.

### Create the logs directories

```bash
mkdir -p /var/log/httpd/foo.com/mkdir -p /var/log/httpd/bar.com/
```

### Restart Apache service

```bash
systemctl restart httpd
```

## Testing

Let's test our configuration now !
Open your browser and access to `http://publicIP` where `publicIP` is one of the public IP addresses created in the previous section.

In my case, [http://20.67.187.123](http://20.67.187.123) leads me to **foo.com** website

![](https://cdn-images-1.medium.com/max/800/1*U1cRwdeiMxqMfa5l9AlaWw.png)

While [http://20.67.188.162](http://20.67.188.162) leads me to **bar.com** website

![](https://cdn-images-1.medium.com/max/800/1*Bu043ax9jC45WyDeEk53_w.png)

Yeaaaah we got it workiing !!

<iframe src="https://giphy.com/embed/l3V0dy1zzyjbYTQQM" width="800" height="400" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/new-girl-fox-new-girl-jeff-day-l3V0dy1zzyjbYTQQM"></a></p>

As you can see, Apache correctly serves the websites according to the IP address. All you need now is to update the DNS records of your domain name to point to the correct public IP address. Then you will be able to access to your website by domain name, like [http://foo.com](http://foo.com) or [http://bar.com](http://bar.com).

---
In this article, we deployed multiple websites on the same Azure server with different IP addresses. To achieve that, we had to avoid a little trap concerning the IP address translation in Azure. Thank you for reading to the end, and see you soon for new articles.
You can reach me on  the following platforms:

- [LinkedIn](https://www.linkedin.com/in/kenneth-koffi-6b1218178/)
- [Medium](https://theko2fi.medium.com)
- [Github](https://github.com/theko2fi)

