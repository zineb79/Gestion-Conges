# Schéma de la base de données

Base **PostgreSQL**, gérée par les migrations Django (8 migrations à ce jour).

## Relations

```
Respo ─────────1:1───── ChefInfo
  │ (chef)
  ├───────1:N───── Employe ─────1:N───── Conge
  │                   │                    │
  │                   └──1:N── HistoriqueSalaire
  │                                │       │
  ├──── modifie_par (N:1) ─────────┘       │
  └──── traite_par (N:1) ──────────────────┘
```

| Relation | Suppression du parent |
|---|---|
| `Employe.chef` → `Respo` | `SET NULL` : l'employé reste, sans chef |
| `Conge.employe` → `Employe` | `CASCADE` |
| `Conge.traite_par` → `Respo` | `SET NULL` : la décision reste dans l'historique |
| `HistoriqueSalaire.employe` → `Employe` | `CASCADE` |
| `HistoriqueSalaire.modifie_par` → `Respo` | `SET NULL` |
| `ChefInfo.user` → `Respo` | `CASCADE` |

## Tables

### `Respo` — utilisateur de l'application
Étend l'utilisateur Django (`AbstractUser`) : identifiant, mot de passe haché, nom, email, `is_active`, `last_login`, `date_joined`.

| Champ | Type | Notes |
|---|---|---|
| `role` | texte | `admin` ou `chef` (par défaut `chef`) |
| `telephone` | texte | facultatif |
| `photo` | image | facultative, stockée dans `media/users/` |

### `ChefInfo`
| Champ | Type | Notes |
|---|---|---|
| `user` | 1:1 → `Respo` | limité aux chefs |
| `service` | texte | service dirigé |

### `Employe`
| Champ | Type | Notes |
|---|---|---|
| `matricule` | texte | **clé primaire** |
| `nom`, `prenom` | texte | |
| `cin` | texte | unique |
| `email` | email | unique |
| `telephone`, `adresse` | texte | facultatifs |
| `dateNaissance` | date | |
| `age` | entier | **calculé** à chaque enregistrement, non modifiable |
| `sexe` | choix | M / F |
| `nationalite` | texte | « Marocaine » par défaut |
| `situation_familiale` | choix | célibataire, marié(e), divorcé(e), veuf / veuve |
| `nombre_enfants` | entier | 0 par défaut |
| `dateEmbauche` | date | ne peut pas être dans le futur ; 18 ans minimum à l'embauche |
| `poste`, `departement` | texte | |
| `echelle` | choix | échelles 1 à 11, ou hors échelle |
| `echelon` | entier | 1 à 10 |
| `indice` | entier | |
| `chef` | N:1 → `Respo` | limité aux chefs |
| `etat` | choix | actif, en congé, retraité, démissionné, décédé, muté |
| `joursCongeRestant` | entier ≥ 0 | 20 par défaut |
| `salaire_actuel` | décimal (10, 2) | mis à jour par l'historique des salaires |
| `created_at`, `updated_at` | horodatage | |

Index : `matricule`, `email`, `etat`, `departement`.

### `Conge`
| Champ | Type | Notes |
|---|---|---|
| `employe` | N:1 → `Employe` | |
| `type_conge` | choix | annuel, maladie, sans solde, maternité, autre |
| `dateDebut`, `dateFin` | date | début ≤ fin ; pas dans le passé à la création |
| `dateDemande` | date | automatique |
| `statut` | choix | en attente, approuvé, refusé, annulé |
| `motif` | texte | facultatif |
| `reponse_chef` | texte | obligatoire en cas de refus |
| `date_reponse` | horodatage | date de la décision |
| `traite_par` | N:1 → `Respo` | qui a décidé |

Index : (`employe`, `dateDemande` décroissante), `statut`, (`dateDebut`, `dateFin`).

Règle de validation : pas de chevauchement avec un autre congé **approuvé** du même employé.

### `HistoriqueSalaire`
| Champ | Type | Notes |
|---|---|---|
| `employe` | N:1 → `Employe` | |
| `ancien_salaire`, `nouveau_salaire` | décimal (10, 2) | |
| `type_modification` | choix | augmentation, diminution, ajustement, promotion, salaire initial |
| `pourcentage_changement` | décimal (5, 2) | **calculé** |
| `motif` | texte | |
| `date_effet` | date | |
| `modifie_par` | N:1 → `Respo` | limité aux admins |

Index : (`employe`, `date_effet` décroissante), `type_modification`.

## Choix de modélisation

- **Les employés ne sont pas des utilisateurs.** Cela reflète le fonctionnement réel : c'est le chef qui saisit et traite les demandes. Ajouter un espace employé consisterait à relier un compte `Respo` à une fiche `Employe`, sans changer le reste du modèle.
- **Le matricule comme clé primaire.** C'est l'identifiant que l'administration utilise réellement, et il apparaît dans les URL (`/employes/M100/`).
- **Des valeurs calculées plutôt que saisies** : âge, ancienneté, durée d'un congé en jours ouvrés, pourcentage d'augmentation. Elles ne peuvent pas devenir fausses.
- **Des montants en `Decimal`**, jamais en nombre à virgule flottante, pour une arithmétique exacte.
- **Les décisions restent dans l'historique** : supprimer le compte d'un chef ne supprime ni ses employés ni les congés qu'il a traités (`SET NULL`).
- **Des index sur les colonnes filtrées** (statut, dates, employé), pour que le tableau de bord reste rapide quand le volume augmente.
