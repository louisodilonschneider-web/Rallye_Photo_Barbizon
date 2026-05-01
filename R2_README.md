# 📷 Rallye Photo Barbizon

> Application web complète pour organiser, gérer et noter un rallye photo scolaire à Barbizon (Forêt de Fontainebleau).

---

## Contexte

6 groupes d'élèves parcourent le village avec une fiche de **10 lieux à photographier** chacun. Chaque photo vaut **10 points** (tous les repères ont la même valeur, y compris le JOKER n°10). L'enseignant valide les photos et attribue les points. Le classement est mis à jour en temps réel.

| Groupe | Thème |
|--------|-------|
| A | L'École des Peintres |
| B | Au Cœur de la Forêt |
| C | Vie de Village |
| D | Lumières & Textures |
| E | Détails & Miniatures |
| F | Contrastes & Temps |

**Score max par groupe : 100 points** (10 photos × 10 pts) + bonus enseignant.

---

## Fonctionnalités

- **Fiche de Route** — 10 repères numérotés par groupe, carte interactive Leaflet avec filtres
- **Dépôt Photos** — formulaire élève, upload Storage, écriture Firestore, vérification post-upload
- **Classement** — temps réel, podium, départage par heure d'arrivée
- **Espace Enseignant** — code `1234`, validation, points, bonus, export PDF

---

## Configuration Firebase

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

## ⚠️ Règles Firebase recommandées

### Problème constaté
La Realtime Database affiche une alerte rouge : **règles publiques** (`.read: true`, `.write: true`).
L'application utilise **Firestore** et **Storage** (pas la Realtime Database), mais il faut sécuriser les trois.

---

### Règles Firestore (console → Firestore → Règles)

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Collection photos : élèves peuvent créer, tout le monde peut lire
    // Seul l'enseignant (via l'app) peut modifier/supprimer
    match /photos/{photoId} {
      allow read: if true;
      allow create: if request.resource.data.keys().hasAll([
        'groupeId','groupeNom','photoIndex','photoLabel',
        'consigne','imageUrl','imagePath','statut','uploadedAt'
      ]);
      // Mise à jour : uniquement les champs de validation
      allow update: if request.resource.data.diff(resource.data).affectedKeys()
        .hasOnly(['statut','points','bonusPoints','remarque','validatedAt','updatedAt']);
      allow delete: if false;
    }

    // Collection groupes : lecture libre, écriture contrôlée
    match /groupes/{groupeId} {
      allow read: if true;
      allow write: if groupeId in ['A','B','C','D','E','F'];
    }
  }
}
```

> **Note** : pour une sécurité maximale en production, remplacez `allow create: if true` par une authentification Firebase Auth avec un token élève.

---

### Règles Firebase Storage (console → Storage → Règles)

```javascript
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    // Seul le dossier photos/ est accessible
    match /photos/{fileName} {
      // Lecture publique (pour affichage dans l'app)
      allow read: if true;
      // Écriture : fichier image uniquement, max 10 Mo
      allow write: if request.resource.size < 10 * 1024 * 1024
                   && request.resource.contentType.matches('image/.*');
    }
    // Tout autre chemin est interdit
    match /{allPaths=**} {
      allow read, write: if false;
    }
  }
}
```

---

### Règles Realtime Database (console → Realtime Database → Règles)

L'application n'utilise PAS la Realtime Database. Fermez-la complètement :

```json
{
  "rules": {
    ".read": false,
    ".write": false
  }
}
```

---

## Flux d'upload — étapes détaillées

```
Élève sélectionne groupe + lieu + photo
         │
         ▼
submitPhoto() déclenché
         │
  ┌──────┴──────┐
  │ Validations │  groupeId, photoIdx, fichier présent, type image, taille < 10 Mo
  └──────┬──────┘
         │
         ▼
[STORAGE] storage.ref("photos/groupeX_photoNN_timestamp.ext").put(file)
   → task.on('state_changed') → barre de progression 0→100%
   → await task → upload complet
   → ref.getDownloadURL() → url publique
         │
         ▼ (si Storage échoue → message d'erreur précis + code Firebase)
         │
[FIRESTORE] db.collection('photos').add({ ...données, imageUrl: url })
   → docRef.id retourné
         │
         ▼ (si Firestore échoue → message + fichier conservé dans Storage)
         │
[VÉRIFICATION] db.collection('photos').doc(docRef.id).get()
   → confirme que le document est bien présent
         │
         ▼
[SUCCÈS] Message de confirmation + reset formulaire + rechargement liste élève
         │
         ▼
[ENSEIGNANT] onSnapshot() reçoit le nouveau document → rendu automatique
```

---

## Flux de validation enseignant

```
onSnapshot(db.collection('photos').orderBy('uploadedAt','desc'))
   → allPhotos[] mis à jour en temps réel
   → renderPendingPhotos() → cartes de validation affichées

Enseignant clique Valider / Refuser
   → validatePhoto(photoId, 'valide'|'refuse')
   → db.collection('photos').doc(photoId).update({ statut, points, bonusPoints, remarque })
   → onSnapshot déclenché → classement recalculé → updateRanking()
```

---

## Logs de debug (console navigateur)

| Préfixe | Signification |
|---------|---------------|
| `[UPLOAD]` | Sélection fichier, déclenchement soumission |
| `[STORAGE]` | Progression upload, URL obtenue, erreurs |
| `[FIRESTORE]` | Écriture document, vérification, erreurs permission |

---

## Structure Firestore

### Collection `photos`

| Champ | Type | Description |
|-------|------|-------------|
| `groupeId` | string | A–F |
| `groupeNom` | string | Nom complet du groupe |
| `photoIndex` | number | 0–9 |
| `photoLabel` | string | Intitulé de la photo |
| `consigne` | string | Consigne associée |
| `bonusDispo` | boolean | Toujours `false` (bonus géré par l'enseignant) |
| `imageUrl` | string | URL publique Firebase Storage |
| `imagePath` | string | Chemin dans Storage |
| `comment` | string | Commentaire élève |
| `lieuRepere` | string | Numéro du repère (1–10) |
| `lieuLabel` | string | Nom du lieu |
| `lieuAddr` | string | Adresse |
| `statut` | string | `en_attente` / `valide` / `refuse` |
| `points` | number | 0–10 (attribués par l'enseignant) |
| `bonusPoints` | number | 0 ou 5 |
| `remarque` | string | Commentaire enseignant |
| `uploadedAt` | timestamp | Horodatage serveur |
| `validatedAt` | timestamp | Date de validation |

### Collection `groupes` (document par groupe A–F)

| Champ | Type | Description |
|-------|------|-------------|
| `arrivee` | string | Heure d'arrivée HH:MM |
| `manualPts` | array[10] | Points manuels par photo |
| `manualBonus` | array[10] | Bonus manuels |
| `updatedAt` | timestamp | Dernière modification |

### Firebase Storage

```
photos/
  groupeA_photo01_1714391234567.jpg
  groupeB_photo03_1714392345678.png
  ...
```

---

## Règles de classement

1. Score = somme des points validés (Firestore) + notes manuelles
2. Tri : score décroissant
3. Ex æquo : heure d'arrivée croissante

---

## Accès

| Profil | Onglets accessibles | Code |
|--------|---------------------|------|
| Élève | Fiche de Route, Dépôt Photos, Classement | — |
| Enseignant | Notation Enseignant | `1234` |

---

## Installation

```bash
# Ouvrir directement dans Chrome / Safari
open rallye-photo-barbizon.html

# Ou déployer sur Firebase Hosting
npm install -g firebase-tools
firebase login
firebase init hosting
firebase deploy
```

---

*Rallye Photo Barbizon — README mis à jour automatiquement*
