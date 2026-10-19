[← Défi 5](defi-05.md) · [Sommaire](../README.md) · **Défi 6**

# Défi 6 — HTTPS refuse la connexion ✅

**Mission :** obtenir une réponse de `https://defi-06:8443/` **en conservant la validation du certificat**, sans désactiver la vérification TLS et sans rien installer sur l'ordinateur.

## Compte rendu

| Élément | Réponse |
|---|---|
| **Symptôme** | `curl https://defi-06:8443/` → `Connected` (TCP ✅), puis `TLS alert, unknown CA` et `SSL certificate problem: unable to get local issuer certificate` (code curl 60). |
| **Hypothèse** | La poignée de main TLS échoue : le certificat du serveur est signé par une autorité absente de la liste de confiance de `poste`. |
| **Preuve** | `openssl s_client -connect defi-06:8443 -servername defi-06` → `subject=CN = defi-06` (le bon serveur) et `issuer=CN = Autorite locale du laboratoire` (une autorité privée). `ls -l /opt/lab/certs/` → `ca.crt` disponible. |
| **Cause** | Le certificat est valide, mais signé par l'**autorité locale du labo**, inconnue du magasin de confiance par défaut (`/etc/ssl/certs/`). |
| **Correction** | Côté **client** : `curl --cacert /opt/lab/certs/ca.crt https://defi-06:8443/`. La vérification reste active, on ajoute seulement l'autorité du labo. |
| **Vérification** | `SSL certificate verify ok`, `subjectAltName: host "defi-06" matched`, `HTTP/1.0 200 OK`, corps `OK defi-06`. |

## La poignée de main TLS, lue dans curl

| Ligne curl | Signification |
|---|---|
| `Client hello` / `Server hello` | Client et serveur s'accordent sur le chiffrement |
| `Certificate` | Le serveur présente son certificat |
| `CAfile: ...` | La liste d'autorités utilisée pour vérifier |
| `unknown CA` (avant) | Signature d'une autorité inconnue → refus |
| `SSL certificate verify ok` (après) | Signature vérifiée grâce à `ca.crt` |
| `subjectAltName ... matched` | Le nom demandé (`defi-06`) correspond au certificat |

## Points d'attention

- **`--cacert` attend un fichier.** Mon premier essai, `curl --cacert defi-06`, a renvoyé `no URL specified!` : curl avait pris `defi-06` pour un nom de fichier. Structure correcte : `curl [option valeur] URL`.
- **Jamais `-k` (`--insecure`).** Il désactive toute vérification : un imposteur serait accepté. `--cacert` ajoute une confiance **précise**, `-k` supprime toute confiance.
- **Permissions** : `server.key` est en `-r--------` (400), lisible par son seul propriétaire. Une clé privée ne se partage pas ; seul le certificat public `ca.crt` est distribué.

## 📂 Dans le dépôt

| Étape | Fichier | Ligne | Ce qui s'y passe |
|---|---|---|---|
| 1 | `Dockerfile` | 16–17 | Création de l'autorité locale : `ca.crt` + `ca.key`, nom « Autorite locale du laboratoire » |
| 2 | `Dockerfile` | 22–24 | Certificat serveur au nom de `defi-06` (`CN` et `subjectAltName`) |
| 3 | `Dockerfile` | 30 | Signature du certificat serveur par l'autorité locale |
| 4 | `Dockerfile` | 32 | Suppression de `ca.key` : plus personne ne peut signer de nouveaux certificats avec cette autorité |
| 5 | `Dockerfile` | 36 | `chmod 400 server.key` : clé privée protégée |
| 6 | `docker-compose.yml` | 66–67 | `LAB_PORT: "8443"` et `LAB_TLS: "1"` pour `defi-06` |
| 7 | `app/server.py` | 50–53 | Si TLS activé : chargement de `server.crt` et `server.key` |

L'autorité n'est jamais ajoutée au magasin de confiance de l'image : c'est volontaire, pour que le défi n'ait qu'une seule cause.

Commandes pour le vérifier soi-même :

```bash
openssl x509 -in /opt/lab/certs/server.crt -noout -subject -issuer -dates   # dans poste : identité, signataire, validité
openssl verify -CAfile /opt/lab/certs/ca.crt /opt/lab/certs/server.crt      # dans poste : la chaîne est valide
grep -n "LAB_TLS\|8443" docker-compose.yml                                  # sur l'hôte
```

## Ce que j'en retiens

- HTTPS ajoute une étape entre TCP et HTTP : la **vérification d'identité** du serveur.
- Un certificat peut être valide mais **non reconnu** : la solution est de fournir la bonne autorité, pas de couper la vérification.
- En production, on utilise une autorité publique (Let's Encrypt) ; une autorité privée sert en interne, et doit être distribuée aux clients.
