
# LeanMass Calculator

Application Android de calcul de la **masse maigre corporelle (LBM)**, développée dans le cadre d'un mini-projet d'audit de sécurité mobile. L'objectif est de concevoir une application fonctionnelle tout en appliquant les bonnes pratiques de sécurité Android.

---

## Fonctionnalités

- **Calcul de la masse maigre (LBM)** : saisie du poids, du taux de graisse corporelle et du genre ; affichage du résultat avec comparaison aux normes (homme ≥ 38 kg, femme ≥ 24 kg)
- **Authentification utilisateur** : inscription (`RegisterActivity`) et connexion (`LoginActivity`) avec politique de mot de passe (`PasswordPolicy`) et validation (`ValidationResult`)
- **Historique des calculs** : liste des résultats passés via `HistoryActivity` et `HistoryAdapter`, stockés dans la table `lbm_history`
- **Configuration externe** : seuils de normalité chargés depuis `assets/config.json` via `ConfigManager`

---

## Architecture

```
com.example.leanmasscalculator/
├── MainActivity
├── ui/
│   ├── auth/
│   │   ├── LoginActivity
│   │   └── RegisterActivity
│   └── history/
│       ├── HistoryActivity
│       └── HistoryAdapter
├── viewmodel/
│   └── MainViewModel          (loadHistory, deleteEntry)
├── data/
│   └── local/
│       ├── AppDatabase        (lbm-db, SQLCipher)
│       ├── CalculationDao
│       └── CalculationEntity  (id, user_id, weight, lbmValue, date)
└── util/
    ├── SessionManager         (EncryptedSharedPreferences)
    ├── KeyManager             (clé SQLCipher)
    ├── ConfigManager          (chargement config.json)
    ├── CalculatorUtil         (formule LBM)
    ├── PasswordPolicy         (règles mot de passe)
    └── ValidationResult       (résultats de validation)
```

---

## Sécurité

La sécurité est le cœur de ce projet. Deux couches de protection sont mises en place pour protéger toutes les données locales.

### 1. Chiffrement des préférences : `EncryptedSharedPreferences`

Les données de session et la clé de base de données sont stockées dans un fichier chiffré unique (`session_enc`), protégé par une `MasterKey` AES-256-GCM via le **Android Keystore System**.

```
session_enc  (EncryptedSharedPreferences)
├── user_id        → SessionManager
├── username       → SessionManager
├── is_logged_in   → SessionManager
└── db_pass_key    → KeyManager  (UUID aléatoire généré à l'installation)
```

- Clés chiffrées : **AES-256-SIV**
- Valeurs chiffrées : **AES-256-GCM**
- Clé maître : **Android Keystore**, non extractible

### 2. Chiffrement de la base de données - `SQLCipher`

La base de données `lbm-db` est entièrement chiffrée via SQLCipher (lib native `.so` incluse pour `arm64-v8a`, `armeabi-v7a`, `x86`, `x86_64`). La clé est générée aléatoirement au premier lancement et jamais stockée en clair.

### Flux de sécurité

```
Premier lancement
└── KeyManager.getDatabaseKey()
    └── SessionManager.getEncryptedPrefs()
        ├── Clé présente ? → retourne la clé existante
        └── Absente       → génère UUID → stocke chiffré → retourne la clé

Ouverture DB
└── AppDatabase (lbm-db) ← clé fournie par KeyManager
    └── SQLCipher déchiffre la base en mémoire
```

---

## Stack technique

| Composant | Technologie | Version |
|---|---|---|
| Langage | Kotlin | — |
| UI | View Binding + Material 3 | — |
| Architecture | MVVM (ViewModel + LiveData) | — |
| Base de données | Room + SQLCipher | — |
| Chiffrement prefs | AndroidX Security Crypto | — |
| Clé maître | Android Keystore AES-256-GCM | — |
| Auth cloud | Firebase Authentication | — |
| Coroutines | Kotlinx Coroutines | 1.7.3 |
| Taille APK | debug | 23.2 MB |
| Architectures | arm64-v8a, armeabi-v7a, x86, x86_64 | — |

---

## Dépendances principales

```gradle
// Sécurité
implementation "androidx.security:security-crypto:1.1.0-alpha06"
implementation "net.zetetic:android-database-sqlcipher:4.5.4"
implementation "androidx.sqlite:sqlite-ktx:2.3.1"

// Base de données
implementation "androidx.room:room-runtime:..."
implementation "androidx.room:room-ktx:..."

// Firebase
implementation "com.google.firebase:firebase-auth-ktx:..."

// Coroutines
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3"
```

---

## Schéma de la base de données

**Table `lbm_history`**

| Colonne | Type | Description |
|---|---|---|
| `id` | INTEGER PK | Identifiant auto-incrémenté |
| `user_id` | TEXT | Référence à l'utilisateur connecté |
| `weight` | REAL | Poids saisi (kg) |
| `lbmValue` | REAL | Masse maigre calculée (kg) |
| `date` | TEXT | Date du calcul |

---

## Installation

```bash
git clone https://github.com/wissal555/LeanMass-MASVS-Hardened.git
```

Ouvrir dans **Android Studio** et lancer sur un appareil ou émulateur (API 23+).

L'APK de démo est disponible ici : [`app-debug.apk`](./app-debug.apk)

---
## Travaux précédents

Audit de sécurité mobile réalisé dans le cadre de ma formation (analyse statique et dynamique, référentiel OWASP MASVS) :
[Rapport d'audit (PDF)](https://drive.google.com/file/d/1oetM2hPRpeBw6k8gNYyZDfFzPYqC7EOl/view?usp=sharing)

## Auteur

Projet réalisé dans le cadre d'un cours de **sécurité des applications mobiles**.  
**EZZAIRI wissal** - ENSA Agadir
