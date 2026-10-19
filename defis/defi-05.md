[← Défi 4](defi-04.md) · [Sommaire](../README.md) · **Défi 5** · [Défi 6 →](defi-06.md)

# Défi 5 — Ça répond… mais en erreur ✅

**Mission :** rétablir une réponse de succès sur `http://defi-05:8000/`. Décider d'abord si la requête a atteint un serveur HTTP.

## Compte rendu

| Élément | Réponse |
|---|---|
| **Symptôme** | `curl http://defi-05:8000/` → `Connected`, puis `HTTP/1.0 503 Service Unavailable`, corps `Configuration manquante : LAB_MESSAGE`. |
| **Hypothèse** | La requête **a atteint l'application** (connexion établie, réponse HTTP reçue). Code 5xx : problème côté serveur, dans sa configuration. |
| **Preuve** | `docker compose logs defi-05` → « prêt : http://0.0.0.0:8000 », puis « Configuration manquante : LAB_MESSAGE », puis la ligne d'accès `172.25.0.2 "GET / HTTP/1.1" 503` (ma requête depuis `poste`). `printenv LAB_MESSAGE` → vide. |
| **Cause** | `DEFI_05_MESSAGE=` est vide dans `.env`. L'application exige ce message et renvoie 503 tant qu'il est vide. |
| **Correction** | `DEFI_05_MESSAGE=Bonjour depuis defi-05` (valeur fictive) dans `.env`, puis `docker compose up -d defi-05`. |
| **Vérification** | `curl http://defi-05:8000/` → `HTTP/1.0 200 OK`, corps `OK defi-05`. |

## Refusé ou 503 : deux mondes différents

| | `Connection refused` (défis 1, 2, 4) | `503` (défi 5) |
|---|---|---|
| Connexion TCP | ❌ Refusée | ✅ Établie |
| Qui répond ? | Le système : personne n'écoute | **L'application elle-même** |
| Où chercher ? | Processus, port, adresse d'écoute | **Configuration** de l'application, logs |

→ Une connexion TCP réussie prouve qu'un programme écoute, **pas qu'il fonctionne correctement**.

## Point d'attention

**Nom de variable ≠ valeur.** Mon premier réflexe a été d'écrire `DEFI_05_MESSAGE=LAB_MESSAGE`. Cela aurait fonctionné (la valeur n'est plus vide), mais c'est le **nom** de la variable dans le conteneur, pas un message. La valeur est ce qui suit le `=`, et elle doit rester **fictive** : jamais de vrai secret dans un `.env` d'exercice.

## 📂 Dans le dépôt

Le trajet du message :

| Étape | Fichier | Ligne | Ce qui s'y passe |
|---|---|---|---|
| 1 | `.env` (copie de `.env.example`) | 4 | `DEFI_05_MESSAGE=` : vide |
| 2 | `docker-compose.yml` | 58 | `LAB_REQUIRE_MESSAGE: "1"` : le message est obligatoire pour `defi-05` |
| 3 | `docker-compose.yml` | 59 | `LAB_MESSAGE: ${DEFI_05_MESSAGE-}` : valeur injectée (vide si absente) |
| 4 | `app/server.py` | 25–26 | Lecture de `LAB_REQUIRE_MESSAGE` et `LAB_MESSAGE` |
| 5 | `app/server.py` | 34–35 | Si obligatoire **et** vide (après `.strip()`) → `503` et « Configuration manquante » |
| 6 | `app/server.py` | 63 | Le même contrôle écrit l'avertissement dans les logs au démarrage |

Commandes pour le vérifier soi-même :

```bash
grep -n "DEFI_05" .env                              # étape 1
grep -n "LAB_MESSAGE\|LAB_REQUIRE" docker-compose.yml  # étapes 2 et 3
docker compose exec defi-05 printenv LAB_MESSAGE    # valeur réellement reçue
docker compose logs --tail 5 defi-05                # avertissement et journal d'accès
```

## Ce que j'en retiens

- Un code **5xx** prouve que la requête est arrivée : on cherche dans l'application, pas dans le réseau.
- Les **logs** contiennent souvent la réponse en clair, et le journal d'accès montre qui a appelé (ici l'IP de `poste`).
- Une configuration obligatoire manquante doit faire échouer le service **visiblement** (503 + log) plutôt que silencieusement.
