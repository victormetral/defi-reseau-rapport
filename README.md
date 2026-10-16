# Rapport — Laboratoire « Pourquoi ça ne répond pas ? »

**Auteur :** Victor Metral · Ada Tech School — Spé DevOps / Opérations et Sécurité, Bloc 2
**Formateur :** Cyril · **Laboratoire :** [cbldev/defi-reseau](https://github.com/cbldev/defi-reseau)

Six services Docker sont volontairement en panne sur un réseau privé.
Pour chacun : **symptôme → hypothèse → preuve → cause → correction → vérification**.
Critère de réussite : `GET /` renvoie `HTTP 200` et le corps `OK defi-NN`.

## Sommaire

| Défi | Titre | Couche en cause | État |
|---|---|---|---|
| [01](defis/defi-01.md) | Personne ne répond | Processus non lancé | ✅ |
| [02](defis/defi-02.md) | La mauvaise porte | — | ⏳ |
| [03](defis/defi-03.md) | Adresse inconnue | — | ⏳ |
| [04](defis/defi-04.md) | Ça marche seulement chez lui | — | ⏳ |
| [05](defis/defi-05.md) | Ça répond… mais en erreur | — | ⏳ |
| [06](defis/defi-06.md) | HTTPS refuse la connexion | — | ⏳ |

Chaque défi contient : le compte rendu, les fausses pistes rencontrées, et un bloc **📂 Dans le dépôt** qui suit le trajet de la cause dans le code du laboratoire.

## Environnement

| Élément | Valeur |
|---|---|
| Machine | MacBook Air, macOS, terminal zsh |
| Docker | Docker Desktop, plugin `docker compose` v2+ |
| Lancement | `cp .env.example .env` → `docker compose build poste` → `docker compose up -d --no-build` |
| Client | `docker compose exec poste bash` |

**Observation préalable :** depuis le Mac, `curl http://defi-01:8000/` échoue avec `Could not resolve host`.
Les noms `defi-01`, `poste`… ne sont connus que du DNS interne de Docker (`127.0.0.11`), à l'intérieur du réseau `atelier`.
Tous les tests client se font donc depuis `poste`.

## Synthèse

| Test réussi | Ce qu'il prouve | Ce qu'il ne prouve pas |
|---|---|---|
| `ping` | La machine est joignable sur le réseau | Qu'une application écoute |
| Connexion TCP | Un programme écoute sur ce port | Qu'il répond correctement |
| `HTTP 200` + bon corps | L'application fonctionne | — |

> « Un ping réussi prouve que la machine est joignable, pas que l'application marche. »

*À compléter après les six défis.*
