# 📷 Rallye Photo Barbizon

> Application web complète pour organiser, gérer et noter un rallye photo scolaire dans le village de Barbizon (Forêt de Fontainebleau).

---

## Contexte du projet

Le Rallye Photo Barbizon est une activité pédagogique dans laquelle **6 groupes d'élèves** parcourent le village des Peintres et la forêt de Fontainebleau avec une fiche de route de **10 photos à réaliser par groupe**. Chaque groupe a un circuit thématique propre :

| Groupe | Thème |
|--------|-------|
| A | L'École des Peintres |
| B | Au Cœur de la Forêt |
| C | Vie de Village |
| D | Lumières & Textures |
| E | Détails & Miniatures |
| F | Contrastes & Temps |

---

## Fonctionnalités

### 📋 Onglet Fiche de Route (public)
- Affichage des 10 photos à réaliser par groupe avec consignes détaillées
- Carte interactive Leaflet/OpenStreetMap avec marqueurs géolocalisés
- Légende colorée par groupe

### 📸 Onglet Dépôt Photos (élèves)
- Sélection du groupe et de la photo concernée
- Affichage automatique de la consigne associée
- Zone glisser-déposer ou sélection de fichier (JPG, PNG, HEIC — max 10 Mo)
- Aperçu de la photo avant envoi + barre de progression de l'upload
- Envoi vers Firebase Storage + enregistrement Firestore
- Affichage des photos déjà déposées par le groupe avec statut

### 🏆 Onglet Classement (public)
- Podium en temps réel (🥇🥈🥉)
- Tableau complet trié par score puis par heure d'arrivée (ex æquo)
- Galerie des photos validées

### 🏅 Onglet Enseignant (protégé — code : 1234)
- Visualisation de toutes les photos déposées, filtrables par groupe
- Validation / Refus avec attribution de points (0–10) et bonus (+5)
- Champ de remarque enseignant par photo
- Saisie de l'heure d'arrivée de chaque équipe
- Grille de notation manuelle complémentaire
- Export PDF par équipe et export PDF global
- Classement automatique avec départage par heure d'arrivée

---

## Architecture technique

```
rallye-photo-barbizon.html   ← Application monofichier (HTML/CSS/JS)
README.md                    ← Ce fichier

Dépendances externes (CDN) :
├── Google Fonts (Playfair Display, Libre Baskerville, Josefin Sans)
├── Leaflet 1.9.4 (carte interactive)
├── Firebase SDK 10.12.2 (compat)
│   ├── firebase-app-compat.js
│   ├── firebase-firestore-compat.js
│   └── firebase-storage-compat.js
└── jsPDF 2.5.1 (export PDF)
```

---

## Configuration Firebase

### Projet Firebase
- **Nom du projet** : rallye-photo-barbizon
- **Project ID** : rallye-photo-barbizon
- **Région** : europe-west1

### Configuration (intégrée dans le HTML)

```javascript
const firebaseConfig = {
  apiKey:            "AIzaSyBkFDZepMedr5Z7hfKx6IzHq2gMzNuxyWk",
  authDomain:        "rallye-photo-barbizon.firebaseapp.com",
  databaseURL:       "https://rallye-photo-barbizon-default-rtdb.europe-west1.firebaseapp.com",
  projectId:         "rallye-photo-barbizon",
  storageBucket:     "rallye-photo-barbizon.firebasestorage.app",
  messagingSenderId: "486043518228",
  appId:             "1:486043518228:web:1b367da7c3627b07d26e01",
  measurementId:     "G-6W4Y1BRZFF"
};
```

---

## Structure Firebase

### Firestore — Collection `photos`

Chaque document = une photo déposée par un élève.

| Champ | Type | Description |
|-------|------|-------------|
| groupeId | string | Identifiant du groupe (A–F) |
| groupeNom | string | Nom complet du groupe |
| photoIndex | number | Index de la photo (0–9) |
| photoLabel | string | Intitulé de la photo |
| consigne | string | Consigne associée |
| bonusDispo | boolean | Bonus +5 pts possible |
| imageUrl | string | URL publique Firebase Storage |
| imagePath | string | Chemin dans Firebase Storage |
| comment | string | Commentaire élève (optionnel) |
| statut | string | en_attente / valide / refuse |
| points | number | Points attribués (0–10) |
| bonusPoints | number | Points bonus (0 ou 5) |
| remarque | string | Remarque de l'enseignant |
| uploadedAt | timestamp | Horodatage serveur Firebase |

### Firestore — Collection `groupes`

Un document par groupe, ID = lettre du groupe (A–F).

| Champ | Type | Description |
|-------|------|-------------|
| arrivee | string | Heure d'arrivée (HH:MM) |
| manualPts | array[10] | Points manuels par photo |
| manualBonus | array[10] | Bonus manuels par photo (bool) |
| updatedAt | timestamp | Dernière modification |

### Firebase Storage

Chemin : `photos/groupe{ID}_photo{NN}_{timestamp}.{ext}`

Exemple : `photos/groupeA_photo01_1714391234567.jpg`

---

## Règles de classement et départage

1. Score total = points validés Firestore + points grille manuelle
2. Tri principal : score décroissant
3. Départage ex æquo : heure d'arrivée croissante
4. Équipes sans heure d'arrivée : classées en dernier

---

## Droits et accès

| Profil | Accès |
|--------|-------|
| Élèves | Fiche de Route, Dépôt Photos, Classement — libres |
| Enseignant | Notation Enseignant — code 1234 |

---

## Règles Firebase recommandées

### Firestore

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /photos/{docId} {
      allow read: if true;
      allow write: if true;
    }
    match /groupes/{groupeId} {
      allow read: if true;
      allow write: if true;
    }
  }
}
```

### Storage

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /photos/{allPaths=**} {
      allow read: if true;
      allow write: if request.resource.size < 10 * 1024 * 1024
                   && request.resource.contentType.matches('image/.*');
    }
  }
}
```

---

## Installation

```bash
# Option 1 : ouvrir directement dans un navigateur
open rallye-photo-barbizon.html

# Option 2 : Firebase Hosting
npm install -g firebase-tools
firebase login
firebase init hosting
firebase deploy
```

---

## Export PDF

- **Par équipe** : bouton "PDF" dans le tableau enseignant → fichier `Rallye_Barbizon_GroupeX.pdf`
- **Global** : bouton "Export PDF global" → fichier `Rallye_Barbizon_Global.pdf`

Contenu : en-tête groupe/score/heure + liste photos + consignes + statuts + points + remarques + images.

---

*Rallye Photo Barbizon — Document généré automatiquement*
