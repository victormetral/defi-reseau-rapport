[← Sommaire](../README.md) · **Défi 1** · [Défi 2 →](defi-02.md)

# Défi 1 — Personne ne répond ✅

**Mission :** obtenir une réponse de `http://defi-01:8000/`. Un collègue affirme que « la machine est bien allumée ».

## Compte rendu

| Élément | Réponse |
|---|---|
| **Symptôme** | `ping -c 2 defi-01` : 2/2 paquets reçus (IP `172.25.0.5`). `curl http://defi-01:8000/` : `Connection refused` en 0 ms. |
| **Hypothèse** | La machine est joignable, mais aucun programme n'écoute sur le port 8000. Deux pistes : (A) serveur sur un autre port, (B) aucun serveur. |
| **Preuve** | `docker compose logs defi-01` → « serveur HTTP désactivé ». `ps -eo args` → seulement `sleep infinity`, aucun processus Python. `ss -lnt` → aucune écoute sur 8000 ; la seule ligne `LISTEN` (`127.0.0.11:44855`) est le DNS interne de Docker. **Hypothèse B confirmée.** |
| **Cause** | `DEFI_01_ACTIF=0` dans `.env`. Compose le transmet au conteneur sous la forme `LAB_ACTIVE=0` ; le script `start.sh` lance alors `sleep infinity` au lieu du serveur. |
| **Correction** | `DEFI_01_ACTIF=1` dans `.env`, puis `docker compose up -d defi-01` (recréation du conteneur). |
| **Vérification** | `curl http://defi-01:8000/` → `HTTP/1.0 200 OK`, corps `OK defi-01`, en-tête `Server: BaseHTTP/0.6 Python/3.13.16`. |

## Fausses pistes rencontrées

**1. Une ligne `LISTEN` qui n'est pas l'application.**
`ss -lnt` affiche une écoute sur `127.0.0.11:44855`. C'est le résolveur DNS que Docker installe dans chaque conteneur, pas le serveur web : aucun processus correspondant dans `ps`, et les logs disent « désactivé ».
→ Ne jamais conclure sur une seule commande : croiser ports, processus et logs.

**2. `restart` ne relit pas le `.env`.**
Après modification du `.env`, `docker compose restart defi-01` laisse les logs inchangés et `printenv LAB_ACTIVE` renvoie encore `0`.
Les variables d'environnement sont fixées **à la création** du conteneur.

| Commande | Relit le `.env` ? |
|---|---|
| `docker compose restart <service>` | Non : relance le même conteneur |
| `docker compose up -d <service>` | Oui : recrée le conteneur si la configuration a changé |
| `docker compose up -d --force-recreate <service>` | Oui : recrée dans tous les cas |

## 📂 Dans le dépôt

Le trajet de la valeur, du fichier de configuration jusqu'au comportement observé :

| Étape | Fichier | Ligne | Ce qui s'y passe |
|---|---|---|---|
| 1 | `.env` (copie de `.env.example`) | 2 | `DEFI_01_ACTIF=0` |
| 2 | `docker-compose.yml` | 31 | `LAB_ACTIVE: ${DEFI_01_ACTIF:-0}` : Compose injecte la valeur dans le conteneur (0 par défaut si absente) |
| 3 | `app/start.sh` | 13–15 | Si `LAB_ACTIVE` vaut `0` : affiche « serveur HTTP désactivé » puis `exec sleep infinity` |
| 4 | `app/start.sh` | 17 | Ligne jamais atteinte : `exec python3 /opt/lab/server.py` |
| 5 | `app/server.py` | — | Jamais lancé, d'où l'absence de processus Python et de port 8000 |

Commandes pour le vérifier soi-même :

```bash
grep -n "DEFI_01" .env                       # étape 1
grep -n "LAB_ACTIVE" docker-compose.yml      # étape 2
sed -n '12,18p' app/start.sh                 # étapes 3 et 4
docker compose exec defi-01 printenv LAB_ACTIVE   # valeur réellement reçue par le conteneur
```

## Ce que j'en retiens

- Un **ping réussi** prouve que la machine est joignable, pas que l'application marche.
- `Connection refused` immédiat = machine joignable, mais **rien n'écoute** sur ce port (≠ `timeout`, qui évoque plutôt un pare-feu).
- Une variable modifiée dans `.env` impose de **recréer** le conteneur, pas seulement de le redémarrer.
