# 📷 Rallye Photo Barbizon

Application web complète pour un rallye photo scolaire à Barbizon.

---

## Projets Firebase

| Projet | Usage |
|--------|-------|
| ~~Rallye Photo Barbizon~~ | Ancien projet (Storage non disponible) |
| **Rallye Photo Barbizon 2** | ✅ Projet actif |

---

## Configuration active

### Firebase (Rallye Photo Barbizon 2)
- **Project ID** : rallye-photo-barbizon-2
- **Firestore** : eur3 (europe-west)
- **Storage** : non utilisé → remplacé par Cloudinary

### Cloudinary (remplace Firebase Storage)
- **Cloud name** : dilm3i5kw
- **Upload preset** : rallye-barbizon (Unsigned)
- **URL upload** : https://api.cloudinary.com/v1_1/dilm3i5kw/image/upload
- **Dossier** : rallye-barbizon/

---

## Architecture

```
Photo élève → Cloudinary (image) + Firestore (métadonnées)
                    ↓
            URL Cloudinary enregistrée dans Firestore
                    ↓
            Enseignant voit la photo via l'URL
```

---

## Flux d'upload

1. Élève choisit groupe + lieu + photo
2. `submitPhoto()` envoie le fichier à Cloudinary via XMLHttpRequest
3. Cloudinary retourne une URL publique
4. L'URL est enregistrée dans Firestore collection `photos`
5. L'onglet enseignant voit le dépôt en temps réel via `onSnapshot`

---

## Règles Firestore

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /photos/{photoId} {
      allow read: if true;
      allow create: if request.resource.data.keys().hasAll([
        'groupeId','imageUrl','imagePath','statut','uploadedAt'
      ]);
      allow update: if request.resource.data.diff(resource.data).affectedKeys()
        .hasOnly(['statut','points','bonusPoints','remarque','validatedAt','updatedAt']);
      allow delete: if false;
    }
    match /groupes/{groupeId} {
      allow read: if true;
      allow write: if groupeId in ['A','B','C','D','E','F'];
    }
  }
}
```

---

## Structure Firestore — Collection `photos`

| Champ | Type | Description |
|-------|------|-------------|
| groupeId | string | A–F |
| photoIndex | number | 0–9 |
| photoLabel | string | Intitulé |
| consigne | string | Consigne |
| imageUrl | string | URL Cloudinary |
| imagePath | string | public_id Cloudinary |
| lieuRepere | string | Numéro repère 1–10 |
| lieuLabel | string | Nom du lieu |
| comment | string | Commentaire élève |
| statut | string | en_attente / valide / refuse |
| points | number | 0–10 |
| bonusPoints | number | 0 ou 5 |
| remarque | string | Note enseignant |
| uploadedAt | timestamp | Horodatage |

---

## Groupes et scores

| Groupe | Thème | Score max |
|--------|-------|-----------|
| A | L'École des Peintres | 100 pts |
| B | Au Cœur de la Forêt | 100 pts |
| C | Vie de Village | 100 pts |
| D | Lumières & Textures | 100 pts |
| E | Détails & Miniatures | 100 pts |
| F | Contrastes & Temps | 100 pts |

Chaque photo vaut **10 pts**. Bonus éventuel attribué par l'enseignant.

---

## Accès

| Profil | Code |
|--------|------|
| Élèves | Libre |
| Enseignant | 1234 |

---

## Pourquoi Cloudinary plutôt que Firebase Storage ?

Firebase Storage nécessite le plan Blaze (carte bancaire) depuis 2024, même pour les petits projets. Cloudinary offre 25 crédits/mois gratuitement sans carte, ce qui est largement suffisant pour un rallye photo scolaire d'une journée.
