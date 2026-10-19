# Rapport — Laboratoire « Pourquoi ça ne répond pas ? »

**Auteur :** Victor Metral · Ada Tech School — Spé DevOps / Opérations et Sécurité, Bloc 2
**Formateur :** Cyril · **Laboratoire :** [cbldev/defi-reseau](https://github.com/cbldev/defi-reseau)

Six services Docker sont volontairement en panne sur un réseau privé.
Pour chacun : **symptôme → hypothèse → preuve → cause → correction → vérification**.
Critère de réussite : `GET /` renvoie `HTTP 200` et le corps `OK defi-NN`.

## Sommaire

| Défi | Titre | Couche en cause | Corrigé côté | État |
|---|---|---|---|---|
| [01](defis/defi-01.md) | Personne ne répond | Processus non lancé | Serveur (`.env`) | ✅ |
| [02](defis/defi-02.md) | La mauvaise porte | Mauvais port | Client | ✅ |
| [03](defis/defi-03.md) | Adresse inconnue | Nom DNS erroné | Client | ✅ |
| [04](defis/defi-04.md) | Ça marche seulement chez lui | Écoute sur `127.0.0.1` | Serveur (`.env`) | ✅ |
| [05](defis/defi-05.md) | Ça répond… mais en erreur | Variable obligatoire vide (503) | Serveur (`.env`) | ✅ |
| [06](defis/defi-06.md) | HTTPS refuse la connexion | Autorité de certification inconnue | Client (`--cacert`) | ✅ |

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

---

## Synthèse

### 1. Où chaque requête s'est arrêtée

| Étape | Défi 1 | Défi 2 | Défi 3 | Défi 4 | Défi 5 | Défi 6 |
|---|---|---|---|---|---|---|
| Nom → IP (DNS) | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| Connexion TCP | ❌ refusée | ❌ refusée | — | ❌ refusée | ✅ | ✅ |
| Poignée de main TLS | — | — | — | — | — | ❌ CA inconnue |
| Réponse HTTP | — | — | — | — | ❌ 503 | — |

Le message d'erreur indique **à quelle étape** la requête s'arrête, jamais **pourquoi** : la cause se prouve ensuite, côté serveur ou côté client.

### 2. Ce que prouve chaque test

| Test réussi | Ce qu'il prouve | Ce qu'il ne prouve pas |
|---|---|---|
| `dig` renvoie une IP | Le nom existe dans l'annuaire | Que la machine répond |
| `ping` | La machine est joignable sur le réseau | Qu'une application écoute (défi 1) |
| Connexion TCP | Un programme écoute sur ce port et cette adresse | Qu'il répond correctement (défi 5) |
| Poignée de main TLS | Le serveur a prouvé son identité | Que l'application fonctionne |
| `HTTP 200` + bon corps | L'application fonctionne | — |

> « Un ping réussi prouve que la machine est joignable, pas que l'application marche. »

### 3. Les messages d'erreur à reconnaître

| Message | Code curl | Signification | Premier réflexe |
|---|---|---|---|
| `Could not resolve host` | 6 | Nom introuvable | Vérifier l'orthographe, `dig` |
| `Connection refused` (immédiat) | 7 | Rien n'écoute sur ce port/cette adresse | `ss -lnt`, `ps`, logs |
| `unable to get local issuer certificate` | 60 | Certificat signé par une autorité inconnue | `openssl s_client`, `--cacert` |
| `503 Service Unavailable` | — | L'application refuse de servir | Logs, variables d'environnement |

### 4. Boîte à outils

| Outil | Rôle |
|---|---|
| `curl -v` | Suivre une requête étape par étape |
| `dig +short` | Isoler l'étape DNS |
| `ss -lnt` | Voir les ports **et les adresses** en écoute |
| `ps -eo args` | Voir les processus réellement lancés |
| `docker compose logs` | Lire ce que l'application dit d'elle-même |
| `printenv` | Vérifier la configuration réellement reçue par le conteneur |
| `openssl s_client` | Examiner un certificat : `subject` et `issuer` |

### 5. Ce que je retiens

1. **Prouver avant de conclure.** Les défis 1 et 2 ont le même symptôme et deux causes différentes. Mes paris par analogie (défis 2 et 3) se sont révélés faux : seules les preuves ont tranché.
2. **Tester l'hypothèse la plus simple d'abord.** Trois pannes sur six venaient de la requête (port, nom, autorité), pas du serveur.
3. **Croiser les sources.** Une ligne `LISTEN` sur `127.0.0.11` n'était pas l'application : ports, processus et logs se vérifient ensemble.
4. **Recréer, pas redémarrer.** Une variable modifiée dans `.env` n'est relue qu'à la recréation du conteneur (`docker compose up -d`), pas par `restart`.
5. **Ne jamais désactiver la sécurité pour « faire marcher ».** `--cacert` ajoute une confiance précise ; `-k` supprime toute vérification.
6. **Tester depuis le point de vue du client.** « Ça marche chez moi » (défi 4) ne prouve rien pour les autres machines.
