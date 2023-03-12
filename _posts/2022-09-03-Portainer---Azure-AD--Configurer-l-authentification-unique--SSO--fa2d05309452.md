---
layout: post
title: "Portainer & Azure AD: Configurer l'authentification unique (SSO)"
image: https://images.unsplash.com/photo-1521791136064-7986c2920216?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1yZWxhdGVkfDN8fHxlbnwwfHx8fA%3D%3D&auto=format&fit=crop&w=600&q=60
tags: [portainer, azure active directory, single sign-on, sso]
categories: [azure]
---

**Portainer** est un projet open-source qui offre une interface graphique (Web UI) pour la gestion simplifiée des conteneurs Docker. Dans cet article, nous verrons comment nous authentifier sur cette interface web en utilisant notre compte Office 365/Microsoft 365. Pour ce faire, il nous faudra configurer l'authentification unique dans **Azure AD**. Ce dernier est un service de gestion des identités et des accès basée sur le cloud. Il fait partie de [Microsoft Entra](https://aka.ms/MicrosoftEntra) et fournit de nombreuses fonctionnalités dont l'authentification multi-facteur, l'accès conditionnel et notamment l' **authentification unique**.
![](https://cdn-images-1.medium.com/max/800/1*AylWrDXLDM7mYSZVaES5Lg.png)

### **Pré-requis**

Pour suivre ce tutoriel, il vous faudra disposer d'une installation fonctionnelle de Portainer et d'un tenant Azure AD.

### Configuration dans Azure AD

Nous allons nous connecter sur notre [portail Azure](https://portal.azure.com) et accéder au service Azure Active Directory.

#### Inscription d'application

![](https://cdn-images-1.medium.com/max/800/1*UHGUmtd2YVh8POMvG9eBxA.png)

Nous allons nommer notre application **Portainer**
![](https://cdn-images-1.medium.com/max/800/1*ByLi6bLGzCto29pRT_edaw.png)

Ici on saisit l'URL pour accéder à notre application Portainer. Pour moi c'est `https://portainer.traefik.me`
![](https://cdn-images-1.medium.com/max/800/1*R7tzXcPVaS4MglbGWBV9nQ.png)

Dès que l'application est inscrite, nous serons redirigés vers l'interface ci-dessous. Il faudra noter l' ID d'application (client) et l'ID de l'annuaire. Nous en aurons besoin plus tard.
![](https://cdn-images-1.medium.com/max/800/1*dPrHKD3zwUdbLC203lzEEA.png)

#### Personnalisation et propriétés

Dans la partie URL de la page d'accueil, on saisit l'URL qui pointe vers notre instance Portainer. Pour moi c'est `https://portainer.traefik.me`
![](https://cdn-images-1.medium.com/max/800/1*rvlSORLReJdIlHOUwgV0sA.png)

#### Ajout d'un secret client

Ici on ajoute un secret d'une durée de validité de 12 mois. Vous pouvez choisir la durée qui vous convient.
![](https://cdn-images-1.medium.com/max/800/1*-qX3hz8pyXLiDq0Pq7Kt7Q.png)

Notons la valeur du secret généré, nous en aurons besoin plus tard
![](https://cdn-images-1.medium.com/max/800/1*eThfriXLneRUWEbn3qj_hA.png)

#### Configuration du jeton

![](https://cdn-images-1.medium.com/max/800/1*knNdXH20Pc9L9yZ1EhOiVA.png)

Ajoutons les revendications **upn** et **email**
![](https://cdn-images-1.medium.com/max/800/1*xmw2oGgVuOoDqmXWp90tHQ.png)![](https://cdn-images-1.medium.com/max/800/1*kx_oLbp3xRP_FkCgUheYmQ.png)![](https://cdn-images-1.medium.com/max/800/1*p8sV9wPaZpuLviGjc7-R-A.png)

Voilà
![](https://cdn-images-1.medium.com/max/800/1*i6dmpFCfctury5wtK17ZGw.png)

#### API autorisées

Dans la section API autorisées, les autorisations suivantes devraient automatiquement être ajoutées. Si ce n'est pas le cas, faites-le manuellement.
![](https://cdn-images-1.medium.com/max/800/1*sngqyqLu2wxdymr2IypkvQ.png)

Ajoutons en plus, l'autorisation **UserAuthenticationMethod.Read**et accordons le consentement d'administrateur.
![](https://cdn-images-1.medium.com/max/800/1*Fkp9UH-9YyVgTL26jQI7kQ.png)

### Configuration de Portainer

![](https://cdn-images-1.medium.com/max/800/1*c46FhztmbzapskGU1H344g.png)

Nous allons saisir les informations suivantes en remplaçant le `TENANT_ID` par l'ID de notre annuaire.

```
authUrl: https://login.microsoftonline.com/TENANT_ID/oauth2/v2.0/authorize
accessTokenUrl: https://login.microsoftonline.com/TENANT_ID/oauth2/v2.0/token
resourceUrl: https://graph.microsoft.com/v1.0/me          
logoutUrl: https://portainer.traefik.me/logout
userIdentifier: userPrincipalName
scopes: profile openid
```

Le **client ID** correspond à l'ID d'application (client) obtenu après l'inscription de notre application dans Azure. Et le **client secret**, correspond à la valeur du secret généré un peu plus tôt.
![Portainer Custom OAuth Azure AD](https://cdn-images-1.medium.com/max/800/1*Ge0CX6gslaOcKJIEXYxMYQ.png) 

### Demo

Maintenant en accédant à notre application Portainer, nous aurons le choix de nous authentifier avec notre compte Microsoft.
![](https://cdn-images-1.medium.com/max/800/1*u6s3cXO3RwE6X3_dmFwBbQ.png)

Après avoir saisi nos identifiants de connexion Microsoft, nous sommes invités à accorder notre consentement à la nouvelle application.
![](https://cdn-images-1.medium.com/max/800/1*Ze0vwpncrLY7I8wfzQW_dA.png)

On accepte et voilà tout !


