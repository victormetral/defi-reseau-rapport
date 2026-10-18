[← Défi 3](defi-03.md) · [Sommaire](../README.md) · **Défi 4** · [Défi 5 →](defi-05.md)

# Défi 4 — Ça marche seulement chez lui ✅

**Mission :** accéder à `http://defi-04:8000/` depuis `poste`. Un collègue assure avoir reçu une réponse en testant depuis la cible elle-même.

## Compte rendu

| Élément | Réponse |
|---|---|
| **Symptôme** | Depuis `poste` : `curl http://defi-04:8000/` → `Connection refused` (code curl 7). Le nom est résolu, la connexion est refusée. |
| **Hypothèse** | Le serveur tourne, mais il n'accepte pas les connexions qui viennent d'une autre machine. |
| **Preuve** | Depuis la cible : `docker compose exec defi-04 curl -s http://127.0.0.1:8000/` → `OK defi-04`. `ss -lnt` → `LISTEN 127.0.0.1:8000`. Le serveur n'écoute que sur la boucle locale. |
| **Cause** | `DEFI_04_ADRESSE=127.0.0.1` dans `.env`. Le serveur n'accepte que les connexions venant de sa propre machine. |
| **Correction** | `DEFI_04_ADRESSE=0.0.0.0` dans `.env`, puis `docker compose up -d defi-04` (recréation du conteneur). |
| **Vérification** | Depuis `poste` : `curl http://defi-04:8000/` → `HTTP/1.0 200 OK`, corps `OK defi-04`. |

## Les deux points de vue

| Test | Depuis la cible (`127.0.0.1`) | Depuis `poste` (`172.25.0.x`) |
|---|---|---|
| Avant correction | ✅ `OK defi-04` | ❌ `Connection refused` |
| Après correction | ✅ | ✅ `OK defi-04` |

**Le collègue avait raison… pour sa machine seulement.** Tester depuis le serveur lui-même ne prouve pas qu'il est joignable par les autres.

## Points d'attention

- **Format de l'adresse** : une IPv4 s'écrit en 4 nombres séparés par des points (`0.0.0.0`). Le port ne fait pas partie de cette variable : il est réglé séparément.
- **Recréer, pas redémarrer** : comme au défi 1, seule une recréation (`up -d`) relit le `.env`.

| Adresse d'écoute | Signification | Joignable depuis `poste` ? |
|---|---|---|
| `127.0.0.1` | Boucle locale : « moi-même » | Non |
| `0.0.0.0` | Toutes les interfaces de la machine | Oui |

## 📂 Dans le dépôt

Le trajet de l'adresse d'écoute :

| Étape | Fichier | Ligne | Ce qui s'y passe |
|---|---|---|---|
| 1 | `.env` (copie de `.env.example`) | 3 | `DEFI_04_ADRESSE=127.0.0.1` |
| 2 | `docker-compose.yml` | 51 | `LAB_BIND: ${DEFI_04_ADRESSE:-127.0.0.1}` : valeur injectée dans le conteneur (127.0.0.1 par défaut) |
| 3 | `app/server.py` | 23 | `bind=env.get("LAB_BIND", "0.0.0.0")` : lit l'adresse d'écoute |
| 4 | `app/server.py` | 49 | `ThreadingHTTPServer((config.bind, config.port), Handler)` : ouvre l'écoute sur cette adresse |

À noter : `server.py` écoute sur `0.0.0.0` par défaut (ligne 10). C'est la configuration Compose qui impose volontairement `127.0.0.1` pour ce défi.

Commandes pour le vérifier soi-même :

```bash
grep -n "DEFI_04" .env                          # étape 1
grep -n "LAB_BIND" docker-compose.yml           # étape 2
docker compose exec defi-04 printenv LAB_BIND   # valeur réellement reçue
docker compose exec defi-04 ss -lnt             # adresse réellement écoutée
```

## Ce que j'en retiens

- « Ça marche chez moi » se vérifie **depuis le point de vue du client**, jamais seulement depuis le serveur.
- Dans `ss -lnt`, on lit l'**adresse** autant que le port : `127.0.0.1` ferme la porte aux autres machines.
- C'est le piège vu en cours : en production, un service derrière un reverse proxy distant doit écouter sur `0.0.0.0`.
