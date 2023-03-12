---
layout: post
title: Réinitialisation simultanée des mots de passe de plusieurs comptes emails via l'API de cPanel
author: Kenneth KOFFI
categories: ["python"]
image: https://cdn-images-1.medium.com/max/800/1*B0DZ2eQahARSKWGENc0sKA.png
tags: [cpanel, python, API]
---

Au cours d'un projet de migration, j'ai été confronté à la problématique de la réinitialisation des mots de passe d'une centaine de boîtes emails gérées par [**cPanel**](https://cpanel.net/). Si vous avez déjà utilisé **cPanel**, vous savez sûrement que son interface graphique ne permet pas de réinitialiser plusieurs mots de passe à la fois. <br>
Pour contourner cette restriction, il m'a fallu me tourner vers son API. J'ai alors décidé de vous faire gagner du temps lors de vos futurs projets de migrations, en vous présentant la démarche adoptée.

#### Prérequis:

Pour suivre ce tutoriel, il vous faudra un compte admin **cPanel** et un ordinateur avec [python3](https://www.python.org/downloads/) installé.

Nous allons commencer par nous connecter à notre portail d'administration cPanel.

#### Générer un API token (jeton d'API)

Dans la page d'accueil de notre instance **cPanel**, on sélectionne **_Manage API Tokens_** dans la section **_SECURITY_**

![](https://cdn-images-1.medium.com/max/800/1*utzV7izBo-6bOoDl9Sb99g.png)

On génère un jeton qu'on va appeler **_token1_**. Vous êtes libres de le nommer comme bon vous semble.

![](https://cdn-images-1.medium.com/max/800/1*ePjmTPIixLRZiU6AQZUW4A.png)

Veuillez noter sa valeur, nous en aurons besoin par la suite. Ça devrait ressembler à quelque comme ceci: `U7HMR63FGY292DQZ4H5BFH16JL`

![](https://cdn-images-1.medium.com/max/800/1*HIgJV776CDGaxM6uT0lLdg.png)

#### Codage d'un script python

J'ai codé le script suivant qui utilisera l'API de cPanel.

```python
import requests,json,argparse,openpyxl 

headers={'Authorization':'cpanel USERNAME:APITOKEN'} 

parser = argparse.ArgumentParser() 

parser.add_argument("-f","--file", help="excel file to read", required=True) 

parser.add_argument("-o","--output", help="file to write errors and results", default="errorslogs.txt", required=False) 

args = parser.parse_args() 

def main():
    # Define variable to load the wookbook 
    wookbook = openpyxl.load_workbook(args.file) 

    # Define variable to read the active sheet: 
    worksheet = wookbook.active 

    with open(args.output, "w", encoding='utf8') as file1: 
        for row in wookbook.worksheets[0].iter_rows(min_row = 2, max_row = worksheet.max_row, max_col = 3): 
            email = row[0].value 
            domain = row[1].value 
            passwd = row[2].value 

            if (email and passwd and domain) is not None:
                result = updatepassword(str(email),str(passwd),str(domain)) 
                file1.write(str(bool(result['status']))+"\t"+result['errors']+"\n")
              
                print(result['status'],end='\t') 
                print(result['errors'],end='\n') 


def updatepassword(email,password,domain): 

    url="https://example.com:2083/execute/Email/passwd_pop?email={}&password={}&domain={}".format(email,password,domain) 
    resp = requests.get(url,headers=headers) 

    # Decode UTF-8 bytes to Unicode, and convert single quotes  
    # to double quotes to make it valid JSON 
    my_json = resp.content.decode('utf8').replace("'", '"') 

    # Load the JSON to a Python list & dump it back out as formatted JSON 
    data = json.loads(my_json) 

    #record the result 
    resultat={'status':data['status']} 
    if not data['errors']: 
        resultat['errors']="null" 
    else: 
        resultat['errors']=data['errors'][0] 
    #return the result 
    return resultat 


if (__name__=="__main__"): 
    main() 
```

Il s'agit d'un script développé en python qui prend deux arguments: `file`et `output`.

* `file` désigne un fichier Excel (.xlsx) contenant la liste des adresses mails et leurs futurs mots de passe
* `output` désigne le fichier de logs. La valeur par défaut est **errorlogs.txt**

Il (le script) est composé de deux fonctions: `updatepassword` et `main`.

`updatepassword` reçoit en paramètre une adresse mail, le nouveau mot de passe et le nom de domaine. Puis elle invoque la fonction [passwd_pop](https://api.docs.cpanel.net/openapi/cpanel/operation/passwd_pop/) de l'API suivi des paramètres cités précédemment, pour effectuer la mise à jour. Voilà la syntaxe de la requête:

```
url="https://example.com:2083/execute/Email/passwd_pop?email={}&password={}&domain={}".format(email,password,domain)
```

Dans le code ci-dessus, **example.com** est le nom de domaine principale de votre instance cPanel. C'est à dire le nom de domaine que vous utilisez pour accéder au portail d'administration.

L'en-tête de la requête est composée du nom d'utilisateur de votre compte admin et du token généré à l'étape précédente . Dans l'exemple suivant, le mon compte admin est `abacus` et mon jeton d'API est `U7HMR63FGY292DQZ4H5BFH16JL`
```
headers={'Authorization':'cpanel abacus:U7HMR63FGY292DQZ4H5BFH16JL'}
```

Après l'exécution de l'opération, la fonction `updatepassword` retourne le résultat.

`main` est la fonction principale du script. Elle parcoure le fichier excel spécifié avec l'argument `- file` lors de l'exécution du script. A chaque ligne, elle appelle la fonction `updatepassword` avec les données de la ligne courante.

#### Création d'un fichier excel

Le fichier Excel (.xlsx) n'est qu'une simple liste contenant les emails et nouveaux mots de passe.

| email |    domain   |    password    |
|:-----:|:-----------:|:--------------:|
| user1 | example.com | KcVdZpZgebLO0a |
| user2 | example.com | KcVdZpZgebLO0a |
| user3 | example.com | WGDt5IYQX7XUKF |
| user4 | example.com | KcVdZpZgebLO0a |
| user5 | example.com | WGDt5IYQX7XUKF |
| user6 | example.com | KcVdZpZgebLO0a |

Dans l'exemple ci-dessus, les adresses mails sont scindées en deux parties. Si vous souhaitez par exemple mettre à jour le mot de passe du compte **_john@company.com_**, les colonnes _email_ et _domain_ recevront les valeurs suivantes:

* email: **john**
* domain: **company.com**

#### Exécution

Vous pouvez récupérer tous ces fichiers depuis mon [dépôt git ici](https://github.com/theko2fi/reset-email-password-cpanel-remotely.git).

Après avoir cloné le dépôt, on accède au dossier qui contient les fichiers et on installe les dépendances.

```bash
pip3 install -r requirements.txt
```

Puis on exécute notre script depuis notre terminal

```bash
python3 ./resetPasswordCpanel.py -f ./emailscpaneltoreset.xlsx
```

J'exécute ce script depuis Linux. Si vous êtes sur **Windows**, pour vous ce sera plutôt :

```
python3 .\resetPasswordCpanel.py -f .\resetemailpasswordcpanel.xlsx
```

Et voilà !!

Je vous remercie d'avoir suivi ce tutoriel jusqu'au bout.
