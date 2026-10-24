# Frontend

## Technologies

| Besoin | Bibliothèque |
|---|---|
| Framework et build | React 19, TypeScript 5.8, Vite 7 |
| Routage | react-router-dom 7 |
| Appels API | Axios (intercepteurs pour le jeton et son renouvellement) |
| Composants | Ant Design 5 (messages, avatars), CSS par page |
| Styles utilitaires | Tailwind CSS 4, daisyUI |
| Icônes | lucide-react, @ant-design/icons, react-icons |
| Dates | dayjs |
| Documents Word | docxtemplater, PizZip, file-saver |

## Pages

| Route | Page | Accès |
|---|---|---|
| `/` | Connexion | public |
| `/Dashboard` | Tableau de bord : travail du jour et analyse filtrable | connecté |
| `/Accueil` | Employés : cartes, recherche, ajout, modification, documents | connecté |
| `/Congé` | Congés : liste, filtres, création, traitement | connecté |
| `/Historique` | Historique des employés | connecté |
| `/Profile` | Profil et photo | connecté |
| `/password` | Changement de mot de passe | connecté |
| `/Stats` | Utilisateurs et services | admin |
| `/Users` | Gestion des comptes | admin |
| `/Register` | Création d'un compte | admin |

Les routes réservées à l'admin sont enveloppées dans `<ProtectedRoute requireAdmin>`. Le backend vérifie de toute façon chaque requête : la protection côté client sert l'expérience, pas la sécurité.

## Session : `AuthContext`

`AuthContext` expose l'utilisateur connecté et les indicateurs `isAuthenticated`, `isAdmin` et `isChef` à toute l'application. Au démarrage, il relit l'utilisateur enregistré puis rafraîchit son profil auprès du serveur.

## Couche d'accès aux données

```
Composant ──▶ service (authService, employeService) ──▶ api (instance Axios) ──▶ API Django
```

- **`api.ts`** : instance Axios unique.
  - **Intercepteur de requête** : ajoute `Authorization: Bearer <jeton>`.
  - **Intercepteur de réponse** : sur une erreur `401`, demande un nouveau jeton avec le jeton de rafraîchissement, puis rejoue la requête. Si le rafraîchissement échoue, l'utilisateur est déconnecté.
  - **`fetchAll()`** : suit la pagination de l'API (`next`) jusqu'à la dernière page, pour que les listes ne soient jamais tronquées à 10 éléments.
- **Services** : une fonction par opération métier (`getDashboard`, `traiterConge`, `annulerConge`, `updateUser`…), avec des types TypeScript identiques aux réponses de l'API.
- **Adresse de l'API** : fournie par `VITE_API_URL`, sans URL écrite en dur.

## Circuit de validation dans l'interface

L'interface ne modifie jamais le statut d'un congé directement :
- **approuver ou refuser** appelle `POST /conges/{id}/traiter/` ;
- **annuler** appelle `POST /conges/{id}/annuler/` ;
- **modifier une demande** n'envoie que les champs réellement changés.

Les messages d'erreur du serveur, comme « L'employé n'a que 3 jours de congé restants », sont affichés tels quels.

## Tableau de bord

- **Filtre de service** (admin) en haut de la page : il s'applique à tout le tableau de bord.
- **Filtres de période et de type** dans la section « Analyse » : ils ne s'appliquent qu'aux graphiques, pas à la liste du travail du jour.
- **Les graphiques** (barre empilée par statut, barres horizontales par type, histogramme mensuel) sont dessinés en HTML et CSS, sans bibliothèque de graphiques. Ils ont des libellés textuels, une légende avec icônes et une description accessible (`aria-label`). Les couleurs ne portent jamais seules l'information.

## Génération des documents

`DocumentService` charge un modèle `.docx` depuis `public/templates/`, remplace les champs `{{...}}` avec docxtemplater, puis télécharge le résultat.

Pour l'**attestation de salaire**, une fenêtre de saisie liste les 8 émoluments et les 6 retenues du modèle officiel. Le brut, le total des retenues et le net sont calculés en direct, et une ligne à 0 reste vide sur le document.

## Thème

Le mode clair ou sombre est un état de l'application, transmis à chaque page. Les nouvelles pages définissent leurs couleurs avec des variables CSS (`--d-surface`, `--d-ink`, `--d-accent`…), redéfinies pour le mode sombre.

## Structure

```
src/
├── App.tsx              routes
├── main.tsx             point d'entrée, Router
├── components/          Navbar, Footer, ProtectedRoute
├── contexts/            AuthContext
├── interfaces/          une page = un .tsx + son .css
└── services/            api, authService, employeService, DocumentService
public/
└── templates/           modèles Word des attestations
```
