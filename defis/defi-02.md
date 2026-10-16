[← Défi 1](defi-01.md) · [Sommaire](../README.md) · **Défi 2** · [Défi 3 →](defi-03.md)

# Défi 2 — La mauvaise porte ✅

**Mission :** obtenir une réponse de l'application annoncée à `http://defi-02:8000/`. L'équipe affirme avoir lancé son serveur.

## Compte rendu

| Élément | Réponse |
|---|---|
| **Symptôme** | `curl http://defi-02:8000/` : `Connection refused` en 1 ms. Le nom est résolu (IP `172.25.0.5`), donc le DNS fonctionne. |
| **Hypothèse** | Rien n'écoute sur le port 8000. Deux pistes : (A) serveur sur un autre port, (B) aucun serveur. Mon premier pari était B. |
| **Preuve** | `docker compose logs defi-02` → « prêt : http://0.0.0.0:8001 ». `ps -eo args` → `python3 /opt/lab/server.py` présent. `ss -lnt` → `LISTEN 0.0.0.0:8001`. **Hypothèse A confirmée, B réfutée.** |
| **Cause** | Le serveur fonctionne, mais il écoute sur le port **8001**. L'adresse annoncée (port 8000) est fausse. |
| **Correction** | Côté **client** uniquement : `curl http://defi-02:8001/`. Aucune modification du serveur ni du `.env`. |
| **Vérification** | `curl http://defi-02:8001/` → `HTTP/1.0 200 OK`, corps `OK defi-02`. |

## Fausse piste rencontrée

**Même symptôme que le défi 1, cause différente.**
J'ai d'abord parié sur « aucun serveur » (hypothèse B), par analogie avec le défi 1. Les preuves ont montré le contraire.

| | Défi 1 | Défi 2 |
|---|---|---|
| Symptôme | `Connection refused` | `Connection refused` |
| `ps` | `sleep infinity` | `python3 /opt/lab/server.py` |
| `ss -lnt` | Rien pour l'application | `LISTEN 0.0.0.0:8001` |
| Cause | Pas de serveur | Mauvais port |
| Ce qu'on corrige | Le serveur (`.env`) | La requête du client |

→ Un même symptôme peut avoir plusieurs causes : on ne conclut jamais par analogie, toujours par preuve.

## 📂 Dans le dépôt

Le trajet du numéro de port :

| Étape | Fichier | Ligne | Ce qui s'y passe |
|---|---|---|---|
| 1 | `docker-compose.yml` | 38 | `LAB_PORT: "8001"` : le port est fixé en dur pour `defi-02` (aucune variable dans `.env`) |
| 2 | `app/server.py` | 11 | `port: int = 8000` : la valeur par défaut, utilisée par les autres cibles |
| 3 | `app/server.py` | 18 | `port = int(env.get("LAB_PORT", "8000"))` : lit `LAB_PORT`, sinon 8000 |
| 4 | `app/server.py` | 49 | `ThreadingHTTPServer((config.bind, config.port), Handler)` : ouvre l'écoute sur `0.0.0.0:8001` |
| 5 | `app/server.py` | 62 | Affiche « prêt : http://0.0.0.0:8001 », la ligne lue dans les logs |

Commandes pour le vérifier soi-même :

```bash
grep -n "LAB_PORT" docker-compose.yml          # étape 1
grep -n "port" app/server.py                   # étapes 2 à 5
docker compose exec defi-02 printenv LAB_PORT  # valeur réellement reçue par le conteneur
```

## Ce que j'en retiens

- `Connection refused` dit **où** ça bloque (rien n'écoute sur ce port), pas **pourquoi**.
- `ss -lnt` montre la vraie porte d'entrée : on lit l'adresse **et** le port de chaque ligne `LISTEN`.
- Parfois, le serveur va bien et c'est la requête qui est fausse : la correction la moins invasive se fait côté client.
