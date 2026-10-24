# Référence de l'API

Toutes les routes sont préfixées par `/api/`. Sauf mention contraire, elles exigent un jeton JWT :

```
Authorization: Bearer <access_token>
```

Les listes sont paginées par 10 (`?page=2`) et renvoient `{count, next, previous, results}`.

**Rôles** : `admin` = administrateur ; `chef` = chef de service, limité à **ses** employés ; `connecté` = admin ou chef.

---

## Authentification et profil — `/auth`

| Méthode | Route | Rôle | Description |
|---|---|---|---|
| POST | `/auth/login/` | public | Connexion ; renvoie l'utilisateur et les jetons `access` + `refresh` |
| POST | `/auth/token/refresh/` | public | Nouveau jeton d'accès à partir du jeton de rafraîchissement (rotation) |
| POST | `/auth/logout/` | connecté | Déconnexion : met le jeton de rafraîchissement en liste noire |
| POST | `/auth/register/` | admin | Crée un compte admin ou chef (le service est requis pour un chef) |
| GET | `/auth/profile/` | connecté | Profil de l'utilisateur connecté |
| PATCH | `/auth/profile/update/` | connecté | Modifie le profil (nom, email, téléphone, photo) |
| POST | `/auth/change-password/` | connecté | Change le mot de passe (ancien mot de passe exigé) |

## Chefs de service

| Méthode | Route | Rôle | Description |
|---|---|---|---|
| GET | `/chefs/` | connecté | Liste des chefs et de leur service |
| PATCH | `/chef/info/` | chef | Le chef met à jour le nom de son service |

## Gestion des comptes — `/admin/users`

| Méthode | Route | Rôle | Description |
|---|---|---|---|
| GET | `/admin/users/?role=chef&is_active=true` | admin | Liste des comptes, filtrable |
| GET | `/admin/users/{id}/` | admin | Détail d'un compte |
| PATCH | `/admin/users/{id}/` | admin | Modifie un compte : identité, rôle, `is_active`, `service` du chef |
| DELETE | `/admin/users/{id}/` | admin | Supprime un compte ; ses employés restent, sans chef |

Règles : un admin ne peut ni supprimer son propre compte, ni se retirer le rôle admin, ni se désactiver. Le dernier administrateur ne peut pas être supprimé. Ces cas renvoient `400`, avec un message explicite.

## Employés — `/employes`

| Méthode | Route | Rôle | Description |
|---|---|---|---|
| GET | `/employes/` | connecté | Liste ; filtres `etat`, `sexe`, `departement`, `poste`, `chef`, `age_min`, `age_max`, `anciennete_min` ; recherche `?search=` ; tri `?ordering=` |
| POST | `/employes/` | connecté | Crée un employé ; s'il est créé par un chef, il est rattaché à ce chef |
| GET | `/employes/{matricule}/` | connecté | Fiche complète, avec congés et historique des salaires |
| PATCH | `/employes/{matricule}/` | connecté | Modifie ; `salaire_actuel`, `joursCongeRestant` et `chef` sont réservés à l'admin |
| DELETE | `/employes/{matricule}/` | connecté | Supprime, sauf si un congé approuvé est à venir |
| POST | `/employes/{matricule}/changer_etat/` | connecté | `{"etat": "retraite"}` : actif, en congé, retraité, démissionné, décédé, muté |
| GET | `/employes/{matricule}/conges/?statut=&annee=` | connecté | Congés de l'employé |
| GET | `/employes/{matricule}/historique_salaire/` | connecté | Historique des salaires |
| GET | `/employes/statistiques/` | connecté | Effectifs, répartition par département et par sexe, âge et ancienneté moyens |

Pour un chef, toutes ces routes ne portent que sur ses employés. Un matricule hors de son périmètre renvoie `404`.

## Congés — `/conges`

| Méthode | Route | Rôle | Description |
|---|---|---|---|
| GET | `/conges/?statut=&type_conge=&employe=&date_debut=&date_fin=` | connecté | Liste filtrable |
| POST | `/conges/` | connecté | Nouvelle demande (toujours créée « en attente ») |
| GET | `/conges/{id}/` | connecté | Détail, avec durée en jours ouvrés |
| PATCH | `/conges/{id}/` | connecté | Modifie une demande **en attente** (sinon `400`) |
| DELETE | `/conges/{id}/` | connecté | Supprime une demande non approuvée |
| POST | `/conges/{id}/traiter/` | chef de l'employé, admin | `{"action": "approuver" \| "refuser", "commentaire": "..."}` ; le commentaire est obligatoire pour un refus |
| POST | `/conges/{id}/annuler/` | chef de l'employé, admin | Annule un congé approuvé et recrédite les jours |
| GET | `/conges/en_attente/` | connecté | Demandes en attente |

Les champs `statut`, `traite_par`, `reponse_chef` et `date_reponse` sont **en lecture seule** : ils ne changent que par `/traiter/` et `/annuler/`.

Exemple de création :

```http
POST /api/conges/
{
  "employe": "M100",
  "type_conge": "annuel",
  "dateDebut": "2026-11-02",
  "dateFin": "2026-11-06",
  "motif": "Voyage familial"
}
```

Erreurs de validation possibles : date de début dans le passé, date de fin avant la date de début, chevauchement avec un congé approuvé, employé hors du périmètre du chef. Le solde insuffisant est vérifié à l'approbation.

## Historique des salaires — `/historique-salaires`

| Méthode | Route | Rôle | Description |
|---|---|---|---|
| GET | `/historique-salaires/?employe=&type_modification=&annee=` | admin | Liste filtrable |
| POST | `/historique-salaires/` | admin | Enregistre une modification ; met à jour le salaire actuel et calcule le pourcentage |
| PATCH / DELETE | `/historique-salaires/{id}/` | admin | Modifie ou supprime une entrée |

## Tableau de bord et statistiques

| Méthode | Route | Rôle | Description |
|---|---|---|---|
| GET | `/dashboard/?range=week\|month\|year&service={id_chef}&type={type}` | connecté | Indicateurs, demandes à traiter, absents, départs à venir, analyse par statut et par type, jours par mois, soldes faibles |
| GET | `/statistics/?range=week\|month\|year` | admin | Comptes, activité de chaque chef (taux d'approbation, délai moyen de réponse), services, comptes à surveiller, employés sans chef |

Pour un chef, `/dashboard/` est limité à son service, quel que soit le paramètre `service`.

Extrait de réponse de `/dashboard/` :

```json
{
  "employes": { "total": 14, "actifs": 14, "absents_aujourdhui": 2 },
  "conges": { "en_attente": 4, "jours_pris_annee": 121 },
  "analyse": {
    "demandes": 13,
    "statuts": { "en_attente": 4, "approuve": 5, "refuse": 1, "annule": 0 },
    "taux_approbation": 83,
    "par_type": [{ "type": "annuel", "label": "Congé Annuel", "total": 7 }]
  },
  "demandes_en_attente": [{ "id": 31, "nom_complet": "Hiba Alaoui", "duree": 5, "jours_restants": 22 }],
  "par_mois": [12, 20, 3, 15, 7, 7, 3, 5, 0, 20, 4, 0]
}
```

## Codes de réponse

| Code | Signification |
|---|---|
| 200 / 201 / 204 | Succès / créé / supprimé |
| 400 | Données invalides ou règle métier non respectée ; le message est renvoyé dans `error` ou par champ |
| 401 | Jeton absent ou expiré : le frontend tente alors un rafraîchissement automatique |
| 403 | Rôle insuffisant (par exemple, un chef sur une route admin) |
| 404 | Ressource inexistante **ou hors du périmètre** de l'utilisateur |
