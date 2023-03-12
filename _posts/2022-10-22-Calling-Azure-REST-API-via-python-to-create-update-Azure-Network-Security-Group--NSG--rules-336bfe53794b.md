---
layout: post
title: Calling Azure REST API via python to create/update Azure Network Security Group (NSG) rules
author: Kenneth KOFFI
categories: [azure, python]
image: https://cdn-images-1.medium.com/max/800/1*iWOw_th908zPnt2T1XJPRw.png
tags: [azure, python, "rest api", network]
---

In this article, i'm going to show you how to create or update the security rules of your Azure Network Security Group (NSG) using python and Azure REST API. I assume you already have an Azure subscription and already know how to create an Azure NSG. There are many articles on the topic on the internet. So let's not waste time and get straight to the point of this tutorial.

The Azure REST API gives us the ability to manage our resources in Azure by sending HTTP requests. But in order to be able to communicate with the API, our python application need to be registered in Azure. To do this, we will connect to the Azure portal and create an application object in our Azure tenant.

### Create an application object in Azure Active Directory (Azure AD)

Log into the Azure portal and select **Azure Active Directory** service.
![](https://cdn-images-1.medium.com/max/800/1*K7BiJNtXO6kfQhwbHym90g.png)

Select **App registrations**
![](https://cdn-images-1.medium.com/max/800/1*D54Z6OGBns_yo3xWdYm8qQ.png)

Select **New registration**
![](https://cdn-images-1.medium.com/max/800/1*NyHhD6aU7ly4jsNz9vrjMA.png)

Type a name and hit the **register** button
![](https://cdn-images-1.medium.com/max/800/1*8Jy9O3gogwnjWu_qsHm01A.png)

After the creation, we're redirected to a page like this. Take note of **Application (client) ID** and **Directory (tenant) ID**, we will put them in our python code later. Next, we'll create a secret for our application.
![](https://cdn-images-1.medium.com/max/800/1*cl3jVhIfxUcYJ9rK36ddSg.png)

The secret is like a password for our application to be authenticated to Azure Active Directory (which is the authentication service).
![](https://cdn-images-1.medium.com/max/800/1*as1mgYZu79uLL3vWqU8Ikw.png)

I will choose a validity period of 12 months for the client secret. Based on your needs, you can select a longer or shorter period.
![](https://cdn-images-1.medium.com/max/800/1*l9eElCPMIZTandcKfab8_Q.png)

**Take note of the value of the secret** after the generation. When you close that page, you won't be able to see it again. We will put it also inside our python code later.
![](https://cdn-images-1.medium.com/max/800/1*IvHTSqPPAs3ihiN3vPDCDQ.png)

If you would rather use command line to achieve all the things we've done in this section, here you go:

```bash
az ad sp create-for-rbac --name [APP_NAME] --password [CLIENT_SECRET]
```

### Assign permissions to our application

After authentication, Azure will check if our application has the necessary permissions to perform the requested task. Since I also like to respect the principle of least privilege, we will create a custom role that will only contain the permission to create/update a security rule in the NSG. We will then assign this new role to our application.

#### Create a custom role

Go to the access control tab of the resource group containing your NSG.
![](https://cdn-images-1.medium.com/max/800/1*dqbnfaVuMdDqcBiH8jx8Gg.png)![](https://cdn-images-1.medium.com/max/800/1*_sIWnDgxTS8-SrT5Pt2yuQ.png)![](https://cdn-images-1.medium.com/max/800/1*Uulqj70VnYEEOqhk0Trx-g.png)

I will name this role "**Custom NSG Contributor**"

![](https://cdn-images-1.medium.com/max/800/1*KIEbRlqs9D5DuNC2gioQ4w.png)

Hit next button to access to the permissions tab.

![](https://cdn-images-1.medium.com/max/800/1*p_HaLMOfAdn-TKVVPW1bPw.png)

Select the **Read** and **Write** permissions to securityRules

![](https://cdn-images-1.medium.com/max/800/1*t0pKeR245wchiM4rQpNTLg.png)

The scope should be populated automatically. If it doesn't, assign it manually
.
![](https://cdn-images-1.medium.com/max/800/1*AHKtSg0YXUJwffIGaTRYtA.png)

Review and create

![](https://cdn-images-1.medium.com/max/800/1*7cedqthKv-iW6dFM8rYajw.png)

#### Assign the custom role to our application

After the creation of our _"_ **_Custom NSG Contributor_** _"_ role, we have to assign it to our _"_ **_Python NSG_** _"_ application object.
![](https://cdn-images-1.medium.com/max/800/1*V6V8A8DDfGOFuzkNzAF-5Q.png)

Search for the role and select it.

![](https://cdn-images-1.medium.com/max/800/1*IeeTNzQXznwI8v8FnGvY6Q.png)![](https://cdn-images-1.medium.com/max/800/1*0Rq6VpB5upXcHA9J5hNjGw.png)![](https://cdn-images-1.medium.com/max/800/1*ESY8QkqeujvS1qihOpAJLQ.png)

We are done with the Azure portal, let's move on to the code of our python application.

### Calling the API via Python

Download the file below or copy its contents into your code editor.

Replace the values of the following variables with those obtained in the previous sections of this tutorial:

* **tenantId**
* **client_id**
* **client_secret**
* **subscriptionId**
* **resourceGroupName**
* **networkSecurityGroupName**
* **securityRuleName**

In this sample code above, i'm creating the rule _"_ **_AllowAnyHTTPInbound_** _"_ with the following properties:

![](https://cdn-images-1.medium.com/max/800/1*clexJAViKt5wlOuFM7flhQ.png)

This rule will allow all incoming traffic on port 80.

If you want to know more about the properties of the security rules, I suggest you consult the API documentation which you can find here :

[Network Security Groups - Create Or Update - Microsoft Learn](https://learn.microsoft.com/en-us/rest/api/virtualnetwork/network-security-groups/create-or-update?tabs=HTTP)

#### Execution

Execute the code from the terminal:

```bash
python3 app.py
```

Et voilààà !! Congratulations, you get it working.

<iframe src="https://giphy.com/embed/l3V0dy1zzyjbYTQQM" width="800" height="400" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/new-girl-fox-new-girl-jeff-day-l3V0dy1zzyjbYTQQM">via GIPHY</a></p>


In this article, we created a security rule in our Network Security Group using the Rest API provided by Azure. Thank you for following this tutorial to the end and see you soon for new articles.
You can reach me on  the following platforms:

- [LinkedIn](https://www.linkedin.com/in/kenneth-koffi-6b1218178/)
- [Medium](https://theko2fi.medium.com)
- [Github](https://github.com/theko2fi)
