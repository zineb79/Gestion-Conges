# Tests

Les tests de l'API utilisent `rest_framework.test.APITestCase` : chaque test crée sa propre base temporaire, avec un admin, deux chefs, un employé par chef et une demande de congé par employé.

```bash
python manage.py test gestion_conge
# Ran 25 tests ... OK
```

## Démarche

Les premiers tests ont été écrits pour **vérifier les droits d'accès**. Lancés sur la première version du code, 10 des 11 tests échouaient : ils ont révélé que le circuit de validation pouvait être contourné, que les chefs voyaient les données des autres services et que le solde pouvait être faussé. Les corrections ont ensuite été faites jusqu'à ce que tous les tests passent. Chaque nouvelle fonctionnalité a reçu ses tests.

## Cas couverts

### Cloisonnement entre chefs
- Un chef ne voit que les congés de ses employés.
- Lire, modifier, supprimer ou traiter le congé d'un autre service renvoie 404.
- Un chef ne peut pas créer de congé pour l'employé d'un autre chef.
- Un chef ne peut pas annuler le congé d'un autre service.

### Circuit de validation
- Envoyer `statut: approuve` par `PATCH` est sans effet.
- L'approbation déduit les jours du solde, et l'admin peut traiter une demande.
- Un congé approuvé ne peut être ni supprimé ni modifié.
- Un congé déjà commencé peut encore être traité.
- Une nouvelle demande ne peut pas commencer dans le passé.
- La durée est calculée en jours ouvrés : un week-end seul ne coûte aucun jour.

### Employés
- Un chef ne peut modifier ni le salaire, ni le solde, ni le chef rattaché.
- Un employé créé par un chef lui est rattaché automatiquement.
- L'admin peut modifier le salaire, et l'historique des salaires met à jour le salaire actuel.
- Nationalité, échelle et situation familiale : enregistrement, libellés et rejet d'une échelle invalide.

### Comptes
- Le rôle par défaut d'un nouveau compte est « chef ».
- L'admin modifie un chef, son service et son statut actif.
- Un admin ne peut ni se retirer ses droits ni supprimer son propre compte.
- Supprimer un chef conserve ses employés, sans chef.
- Un chef n'a pas accès à la gestion des comptes.
- Le service du chef est bien renvoyé par l'API.

### Tableau de bord et statistiques
- Le tableau de bord est limité au service du chef, même s'il passe un autre service en paramètre.
- Les filtres de service, de période et de type sont appliqués correctement.
- Les absents du jour sont bien détectés.
- Les statistiques des comptes, des chefs (taux d'approbation, délai de réponse) et des services sont exactes, et réservées à l'admin.

## Pistes

- Tests du frontend (Vitest + Testing Library) sur le circuit de validation et la génération des documents.
- Intégration continue : lancer les tests à chaque push avec GitHub Actions.
