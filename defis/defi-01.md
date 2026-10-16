# Rapport — Laboratoire « Pourquoi ça ne répond pas ? »

**Auteur :** Victor Metral
**Formation :** Ada Tech School — Spé DevOps / Opérations et Sécurité, Bloc 2
**Formateur :** Cyril
**Laboratoire :** [cbldev/defi-reseau](https://github.com/cbldev/defi-reseau)

---

## Contexte

Six services Docker sont volontairement en panne sur un réseau privé (`atelier`).
Pour chacun, la démarche suivie est la même :

> **symptôme → hypothèse → preuve → cause → correction → vérification**

- **Client** : conteneur `poste` (`docker compose exec poste bash`).
- **Hôte** : terminal du Mac, pour les logs, l'examen des cibles et les corrections.
- **Critère de réussite** : `GET /` renvoie `HTTP 200` et le corps `OK defi-NN`.

## Environnement

| Élément | Valeur |
|---|---|
| Machine | MacBook Air, macOS, terminal zsh |
| Docker | Docker Desktop, plugin `docker compose` v2+ |
| Lancement | `cp .env.example .env` → `docker compose build poste` → `docker compose up -d --no-build` |
| État initial | 7 conteneurs `Up`, aucun port publié vers l'hôte (réseau `internal: true`) |

**Observation préalable :** lancées depuis le Mac, les commandes `ping defi-01` et `curl http://defi-01:8000/` échouent avec `Could not resolve host`.
Les noms `defi-01`, `poste`… ne sont connus que du **DNS interne de Docker** (`127.0.0.11`), à l'intérieur du réseau `atelier`.
Tous les tests client se font donc depuis `poste`.

---

## Défi 1 — Personne ne répond ✅

**Mission :** obtenir une réponse de `http://defi-01:8000/`. Un collègue affirme que « la machine est bien allumée ».

| Élément | Réponse |
|---|---|
| **Symptôme** | `ping -c 2 defi-01` : 2/2 paquets reçus (IP `172.25.0.5`). `curl http://defi-01:8000/` : `Connection refused` en 0 ms. |
| **Hypothèse** | La machine est joignable, mais aucun programme n'écoute sur le port 8000. Deux pistes : (A) serveur sur un autre port, (B) aucun serveur. |
| **Preuve** | `docker compose logs defi-01` → « serveur HTTP désactivé ». `ps -eo args` → seulement `sleep infinity`, aucun processus Python. `ss -lnt` → aucune écoute sur 8000 ; la seule ligne `LISTEN` (`127.0.0.11:44855`) est le DNS interne de Docker. **Hypothèse B confirmée.** |
| **Cause** | `DEFI_01_ACTIF=0` dans `.env`. Compose le transmet au conteneur sous la forme `LAB_ACTIVE=0` ; le script `start.sh` lance alors `sleep infinity` au lieu du serveur. |
| **Correction** | `DEFI_01_ACTIF=1` dans `.env`, puis `docker compose up -d defi-01` (recréation du conteneur). |
| **Vérification** | `curl http://defi-01:8000/` → `HTTP/1.0 200 OK`, corps `OK defi-01`, en-tête `Server: BaseHTTP/0.6 Python/3.13.16`. |

### Fausse piste rencontrée

Après modification du `.env`, un `docker compose restart defi-01` **ne suffit pas** :
les logs affichent toujours « serveur HTTP désactivé » et `printenv LAB_ACTIVE` renvoie encore `0`.

Les variables d'environnement sont fixées **à la création** du conteneur.
`restart` relance le même conteneur sans relire le `.env` ; `up -d` compare la configuration et **recrée** le conteneur si elle a changé.

| Commande | Relit le `.env` ? |
|---|---|
| `docker compose restart <service>` | Non |
| `docker compose up -d <service>` | Oui, recrée si la configuration a changé |
| `docker compose up -d --force-recreate <service>` | Oui, recrée dans tous les cas |

### Ce que j'en retiens

- Un **ping réussi** prouve que la machine est joignable, pas qu'une application répond.
- `Connection refused` immédiat = machine joignable, mais **rien n'écoute** sur ce port (≠ `timeout`, qui évoque plutôt un pare-feu).
- Ne jamais conclure sur une seule commande : **croiser** logs, processus (`ps`) et ports (`ss`).

---

## Défi 2 — La mauvaise porte

*À venir.*

## Défi 3 — Adresse inconnue

*À venir.*

## Défi 4 — Ça marche seulement chez lui

*À venir.*

## Défi 5 — Ça répond… mais en erreur

*À venir.*

## Défi 6 — HTTPS refuse la connexion

*À venir.*

---

## Synthèse

*À rédiger après les six défis : ce que prouvent, et ne prouvent pas, une réponse au ping, une connexion TCP et une réponse HTTP.*
