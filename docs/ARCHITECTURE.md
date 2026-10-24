# Architecture

## Vue d'ensemble

L'application est découpée en deux parties indépendantes qui communiquent uniquement par une API REST en JSON :

- **Frontend** : application React monopage (SPA). Elle affiche les données, gère la session et génère les documents Word dans le navigateur.
- **Backend** : API Django REST Framework. Elle porte toutes les règles métier, tous les contrôles d'accès, et parle à PostgreSQL.

Le frontend ne prend aucune décision qui engage les données : il demande, le serveur vérifie et décide. Masquer un bouton améliore l'interface, mais ne protège rien.

```
Navigateur (React)  ──HTTP/JSON──▶  API Django  ──ORM──▶  PostgreSQL
      │                                  │
      └─ modèles .docx (public/)         └─ JWT : accès 30 min, rafraîchissement 2 h (rotation + liste noire)
```

## Les couches du backend

| Couche | Fichiers | Responsabilité |
|---|---|---|
| Routage | `Backend/urls.py`, `gestion_conge/urls.py` | Associe chaque URL à une vue ; `DefaultRouter` génère les routes des ViewSets |
| Vues | `api_views.py`, `api_views_employe.py` | Reçoivent la requête, choisissent le serializer, appellent le modèle |
| Permissions | `permissions.py` | Qui peut faire quoi : rôle, puis droits sur l'objet |
| Serializers | `serializers.py`, `serializers_employe.py` | Validation des entrées, forme des réponses, champs en lecture seule |
| Modèles | `models.py` | Données et règles métier (`approuver`, `refuser`, `annuler`, `duree`, `clean`) |
| Filtres | `filters.py` | Filtres de recherche réutilisables (django-filter) |

### Pourquoi les règles métier sont dans les modèles

Un congé peut être approuvé depuis la page Congés, depuis le tableau de bord ou depuis l'admin Django. Si la règle (vérifier le solde, déduire les jours, enregistrer le décideur) était écrite dans une vue, il faudrait la recopier, et une copie finirait par diverger. Elle est donc dans `Conge.approuver()`, et toutes les entrées passent par là.

## Rôles et périmètre

| | Admin | Chef de service |
|---|---|---|
| Employés | tous | uniquement les siens |
| Congés | tous | uniquement ceux de ses employés |
| Traiter un congé | oui | oui, pour ses employés |
| Salaire, solde de congés, chef rattaché | modifiables | lecture seule |
| Historique des salaires | oui | non |
| Comptes utilisateurs, statistiques | oui | non |

Le périmètre est appliqué à **trois niveaux** :
1. **la permission de la vue** : le rôle autorise-t-il cette action ?
2. **le queryset** : un chef ne « voit » que ses données. Un objet hors périmètre renvoie 404, pas 403 ;
3. **la permission par objet** : une seconde vérification sur l'objet précis.

## Le circuit de validation d'un congé

```
            créer                    traiter (approuver)
  (rien) ─────────▶ en attente ────────────────────────▶ approuvé ──annuler──▶ annulé
                        │                                    (jours déduits)     (jours recrédités)
                        └──── traiter (refuser + motif) ───▶ refusé
```

- Le champ `statut` est **en lecture seule** dans l'API. Il ne change que par `POST /conges/{id}/traiter/` ou `POST /conges/{id}/annuler/`.
- Une demande ne peut être modifiée ou supprimée que tant qu'elle est en attente. Un congé approuvé doit être annulé, pour que les jours soient recrédités.
- Un refus exige un motif.

## Cycle de vie d'une approbation

```
POST /api/conges/42/traiter/  {"action": "approuver", "commentaire": "Bon congé"}

1. JWTAuthentication       lit le jeton Bearer → request.user
2. CanApproveLeave         has_permission : rôle chef ou admin ?
3. CongeViewSet.get_object()
     └─ get_queryset()     chef → Conge.objects.filter(employe__chef=user)
     └─ has_object_permission : chef de l'employé, ou admin ?
4. CongeActionSerializer   action valide ? motif présent si refus ?
5. Conge.approuver(chef)   ── @transaction.atomic ──────────────────────
     a. statut == en_attente ?
     b. employe = Employe.objects.select_for_update().get(...)   ← verrou
     c. duree = jours ouvrés entre dateDebut et dateFin
     d. solde suffisant ?  sinon ValidationError
     e. employe.joursCongeRestant -= duree
     f. statut = approuve, traite_par = chef, date_reponse = maintenant
   ── fin de transaction : tout est enregistré, ou rien ────────────────
6. ← 200 {"message": "Congé approuvé avec succès", "conge": {...}}
```

### Atomicité

La déduction des jours (table `Employe`) et le changement de statut (table `Conge`) forment **une seule opération**. Sans transaction, une erreur entre les deux laisserait un solde diminué pour un congé toujours en attente. Avec `@transaction.atomic`, PostgreSQL annule tout en cas d'échec.

### Concurrence

Deux approbations simultanées pour le même employé liraient le même solde (par exemple 10 jours), et accorderaient chacune leur congé (8 puis 6 jours). `select_for_update()` verrouille la ligne de l'employé jusqu'à la fin de la transaction : la seconde approbation attend, relit le solde à jour (2 jours) et refuse.

## Jours ouvrés

La durée d'un congé compte les jours **du lundi au vendredi**, bornes incluses. Un congé du vendredi au lundi coûte 2 jours, pas 4. Le même calcul existe côté client pour l'aperçu de la saisie, afin que l'écran affiche le chiffre qui sera réellement déduit.

## Tableau de bord et statistiques

Deux points d'API dédiés font les calculs côté serveur, au lieu de télécharger toutes les données dans le navigateur :

- `GET /api/dashboard/` : congés et indicateurs, avec un périmètre automatique (chef = son service) et des filtres (service pour l'admin, période, type) ;
- `GET /api/statistics/` : comptes, activité de chaque chef (demandes reçues, traitées, taux d'approbation, délai moyen de réponse) et services. Réservé à l'admin.

## Génération des documents

Les attestations sont générées **dans le navigateur** à partir de modèles Word (`public/templates/*.docx`) :

1. le modèle est chargé avec `fetch` ;
2. docxtemplater remplace les champs `{{nom}}`, `{{matricule}}`, `{{brut_mensuel}}`… ;
3. le fichier est téléchargé avec file-saver.

L'administration peut ainsi modifier la mise en page de ses documents dans Word, sans toucher au code. Les totaux de l'attestation de salaire (brut, retenues, net) sont calculés par l'application, pas saisis.

## Configuration

Toutes les valeurs sensibles ou propres à l'environnement sont lues dans un fichier `.env` (`python-dotenv`) : clé secrète Django, mode debug, hôtes autorisés, base de données, origines CORS. Côté frontend, l'adresse de l'API est fournie par `VITE_API_URL`. Voir [`.env.example`](../.env.example).

## Compromis et améliorations prévues

- **Jours fériés** : le décompte exclut les week-ends, pas encore les jours fériés. Une table des jours fériés, gérée par l'admin, compléterait `duree()`.
- **Jeton dans le localStorage** : simple à mettre en place, mais exposé en cas de faille XSS. Un cookie HTTP-only pour le jeton de rafraîchissement serait plus sûr.
- **Détail de la paie** : l'application ne stocke que le salaire global ; les indemnités et retenues de l'attestation de salaire sont saisies au moment de la génération. Une table de paie mensuelle permettrait de les pré-remplir.
- **État « en congé »** : l'état d'un employé n'est pas encore mis à jour automatiquement au début et à la fin d'un congé ; le tableau de bord calcule les absents à partir des dates.
