[← Défi 2](defi-02.md) · [Sommaire](../README.md) · **Défi 3** · [Défi 4 →](defi-04.md)

# Défi 3 — Adresse inconnue ✅

**Mission :** le ticket indique `http://defi-3:8000/`. La requête n'obtient pas de réponse HTTP : déterminer à quelle étape elle s'arrête et retrouver l'adresse utilisable.

## Compte rendu

| Élément | Réponse |
|---|---|
| **Symptôme** | `curl http://defi-3:8000/` : `Could not resolve host: defi-3` (code curl 6). Aucune ligne `Trying` : aucune adresse IP obtenue, donc aucune connexion tentée. |
| **Hypothèse** | La requête s'arrête à la résolution de nom (DNS). Deux pistes : (A) le nom du ticket est faux, (B) le DNS de Docker est en panne. Mon premier pari était B. |
| **Preuve** | `dig +short defi-3` → aucune réponse. `dig +short defi-03` → `172.25.0.5`. Le DNS répond pour un nom correct. **Hypothèse A confirmée, B réfutée.** |
| **Cause** | Faute dans le ticket : `defi-3` au lieu de `defi-03`. Pour le DNS, ce sont deux noms totalement différents. |
| **Correction** | Côté **client** uniquement : `curl http://defi-03:8000/`. Aucune modification du serveur ni du `.env`. |
| **Vérification** | `curl http://defi-03:8000/` → `HTTP/1.0 200 OK`, corps `OK defi-03`. |

## Fausse piste rencontrée

**Accuser le système avant la requête.**
J'ai d'abord parié sur une panne du DNS (B). Une seule commande suffisait à trancher : si `dig defi-03` répond, l'annuaire fonctionne.

→ Défis 2 et 3 : même schéma. L'hypothèse la plus simple (une erreur dans la requête) se teste **avant** l'hypothèse lourde (un service en panne).

## Où la requête s'arrête

| Étape | Défi 1 | Défi 2 | Défi 3 |
|---|---|---|---|
| 1. Nom → IP (DNS) | ✅ | ✅ | ❌ `Could not resolve host` |
| 2. Connexion TCP au port | ❌ `Connection refused` | ❌ `Connection refused` | — jamais atteinte |
| 3. Réponse HTTP | — | — | — |

## 📂 Dans le dépôt

D'où vient le nom `defi-03` :

| Étape | Fichier | Ligne | Ce qui s'y passe |
|---|---|---|---|
| 1 | `docker-compose.yml` | 40 | `defi-03:` : le **nom du service** Compose |
| 2 | `docker-compose.yml` | 5 | `networks: [atelier]` : chaque service rejoint le réseau `atelier` |
| 3 | `docker-compose.yml` | 69–71 | Réseau `atelier`, `internal: true` : privé, sans accès extérieur |
| 4 | (Docker) | — | Sur ce réseau, le DNS interne de Docker (`127.0.0.11`) enregistre chaque **nom de service**. Aucun service ne s'appelle `defi-3` |

Le `LAB_NAME: defi-03` (ligne 44) ne sert qu'au texte de la réponse (`OK defi-03`) : ce n'est pas lui qui crée le nom DNS.

Commandes pour le vérifier soi-même :

```bash
docker compose config --services     # liste les noms de services = noms DNS disponibles
dig +short defi-03                   # dans poste : nom connu → IP
dig +short defi-3                    # dans poste : nom inconnu → rien
```

## Ce que j'en retiens

- `Could not resolve host` = la requête s'arrête **avant** le réseau : on vérifie d'abord l'orthographe du nom.
- `dig` interroge l'annuaire seul, sans HTTP ni port : c'est l'outil pour isoler l'étape DNS.
- Dans Docker Compose, le **nom du service** est le nom DNS de la machine.
