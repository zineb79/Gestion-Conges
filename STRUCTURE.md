# Structure du projet

L'arborescence ci-dessous reflète le vrai code. Le code source est privé ; ce document montre comment le projet est organisé.

## Backend — Python / Django REST Framework

```
Backend/
├── manage.py
├── requirements.txt
├── .env.example                    # variables d'environnement (noms uniquement)
├── Backend/                        # configuration du projet Django
│   ├── settings.py                 # lit les secrets dans .env (python-dotenv)
│   ├── urls.py                     # /admin/ et /api/
│   ├── wsgi.py
│   └── asgi.py
└── gestion_conge/                  # application métier
    ├── models.py                   # Respo, ChefInfo, Employe, Conge, HistoriqueSalaire
    │                               #   + règles : approuver(), refuser(), annuler(), duree()
    ├── serializers.py              # comptes : inscription, connexion, profil, gestion admin
    ├── serializers_employe.py      # employés (liste / détail / écriture), congés, salaires
    ├── permissions.py              # IsAdmin, CanManageEmployees, CanApproveLeave, ...
    ├── api_views.py                # authentification, comptes, tableau de bord, statistiques
    ├── api_views_employe.py        # EmployeViewSet, CongeViewSet, HistoriqueSalaireViewSet
    ├── filters.py                  # filtres django-filter
    ├── urls.py                     # routes /api/... + DefaultRouter
    ├── admin.py                    # administration Django
    ├── tests.py                    # 25 tests APITestCase
    └── migrations/                 # 0001 → 0008
```

## Frontend — React / TypeScript / Vite

```
Frontend/
├── index.html
├── package.json
├── vite.config.ts
├── tsconfig*.json
├── .env.example                    # VITE_API_URL, ...
├── public/
│   └── templates/
│       ├── Attestation_Travail.docx
│       └── Attestation_Salaire.docx
└── src/
    ├── main.tsx                    # point d'entrée + BrowserRouter
    ├── App.tsx                     # routes, thème
    ├── components/
    │   ├── Navbar.tsx / .css
    │   ├── footer.tsx / .css
    │   └── ProtectedRoute.tsx      # connexion et rôle requis
    ├── contexts/
    │   └── AuthContext.tsx         # utilisateur, isAdmin, isChef
    ├── interfaces/                 # une page = .tsx + .css
    │   ├── auth.tsx                # connexion
    │   ├── dashboard.tsx           # tableau de bord + analyse filtrable
    │   ├── accueil.tsx             # employés, attestations
    │   ├── Congé.tsx               # congés
    │   ├── historique.tsx
    │   ├── Stats.tsx               # utilisateurs et services
    │   ├── users.tsx               # gestion des comptes
    │   ├── register.tsx
    │   ├── profile.tsx
    │   └── password.tsx
    ├── services/
    │   ├── api.ts                  # Axios, intercepteurs, fetchAll()
    │   ├── authService.ts          # session et comptes
    │   ├── employeService.ts       # employés, congés, tableau de bord
    │   └── DocumentService.ts      # attestations Word
    └── assets/                     # logos, icônes
```
