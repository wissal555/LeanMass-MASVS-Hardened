# LeanMass Calculator

Application Android de calcul de la **masse maigre corporelle (LBM)**, développée en Kotlin dans le cadre du module Développement Mobile, puis auditée et sécurisée dans le cadre d'un mini-projet de sécurité mobile (ENSA Agadir, filière Sécurité IT et Confiance Numérique, 2025-2026, encadré par Pr. Amrani).

Le projet a été mené en **trois temps** : développer l'application, l'auditer selon l'OWASP MASVS, puis corriger les vulnérabilités les plus critiques. Cette version est **durcie sur un périmètre précis** (stockage local, cryptographie, authentification) : voir [Couverture MASVS](#couverture-masvs).

---

## Démarche

| Phase | Objectif | Résultat |
|---|---|---|
| 1. Développement | Concevoir une application fonctionnelle (calcul, authentification, historique) | Version initiale |
| 2. Audit | Revue du code source selon l'OWASP MASVS | [Rapport d'audit (PDF)](https://drive.google.com/file/d/1oetM2hPRpeBw6k8gNYyZDfFzPYqC7EOl/view?usp=sharing) : 5 vulnérabilités identifiées |
| 3. Hardening | Corriger les 3 vulnérabilités les plus critiques (V1, V2, V3) | Version durcie (ce dépôt) |

---

## Fonctionnalités

- **Calcul de la masse maigre (LBM)** avec la formule de Boer (homme et femme), et retour visuel immédiat (satisfaisant / à surveiller) par rapport aux normes (homme ≥ 38 kg, femme ≥ 24 kg).
- **Authentification** par email et mot de passe via Firebase Authentication : inscription (`RegisterActivity`) et connexion (`LoginActivity`), avec politique de mot de passe (`PasswordPolicy`) et validation (`ValidationResult`).
- **Historique** des calculs avec consultation (`HistoryActivity`, `HistoryAdapter`) et suppression d'une entrée.
- **Double persistance** : base locale SQLite (Room) et base cloud Firebase Firestore.
- **Configuration externe** : seuils de normalité chargés depuis `assets/config.json` via `ConfigManager`.

---

## Phase 1 : Développement

### Stack technique

| Composant | Technologie |
|---|---|
| Plateforme | Android (API 24+) |
| Langage | Kotlin 1.9.10 |
| Architecture | MVVM (ViewModel + LiveData) |
| Base de données locale | Room (SQLite) |
| Base de données cloud | Firebase Firestore |
| Authentification | Firebase Authentication |
| Interface | ViewBinding, CardView, RecyclerView |
| Build | Gradle (KTS), AGP 8.2.0 |
| Asynchronisme | Kotlinx Coroutines 1.7.3 |
| Architectures supportées | arm64-v8a, armeabi-v7a, x86, x86_64 |

### Architecture

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
│       ├── AppDatabase        (lbm-db)
│       ├── CalculationDao
│       └── CalculationEntity  (id, user_id, weight, lbmValue, date)
└── util/
    ├── SessionManager         (session utilisateur)
    ├── KeyManager             (clé de la base)
    ├── ConfigManager          (chargement de config.json)
    ├── CalculatorUtil         (formule LBM)
    ├── PasswordPolicy         (règles de mot de passe)
    └── ValidationResult       (résultats de validation)
```

### Schéma de la base de données locale

**Table `lbm_history`**

| Colonne | Type | Description |
|---|---|---|
| `id` | INTEGER PK | Identifiant auto-incrémenté |
| `user_id` | TEXT | Utilisateur connecté |
| `weight` | REAL | Poids saisi (kg) |
| `lbmValue` | REAL | Masse maigre calculée (kg) |
| `date` | TEXT | Date du calcul |

---

## Phase 2 : Audit de sécurité

L'audit a été réalisé selon l'**OWASP MASVS**, par **revue du code source** de l'application avant toute modification. Cinq catégories ont été examinées : STORAGE, CRYPTO, AUTH, NETWORK et CODE.

Le détail des constats, de leur impact et de leur criticité se trouve dans le [rapport d'audit (PDF)](https://drive.google.com/file/d/1oetM2hPRpeBw6k8gNYyZDfFzPYqC7EOl/view?usp=sharing).

### Principaux constats

| Réf. | Constat (version initiale) | Catégorie MASVS | Criticité | Traité |
|---|---|---|---|---|
| V1 | Session (`userId`, `username`) stockée en clair dans les SharedPreferences | MASVS-STORAGE-1 | Haute | Oui : `EncryptedSharedPreferences` (`SessionManager`) |
| V2 | Base Room (`lbm-db`) non chiffrée, données de santé en clair | MASVS-CRYPTO-1 | Haute | Oui : SQLCipher + `KeyManager` |
| V3 | Mot de passe : 6 caractères minimum, aucun contrôle de complexité, pas de verrouillage côté application | MASVS-AUTH-1 | Moyenne | Oui : `PasswordPolicy` ; verrouillage assuré côté serveur par Firebase |
| V4 | Clé API Firebase en clair dans `google-services.json`, extractible de l'APK | MASVS-NETWORK-1 | Moyenne | Non : documenté (dépend de la configuration Firebase) |
| V5 | `e.printStackTrace()` dans `ConfigManager` et `FirebaseManager` | MASVS-CODE-1 | Faible | Non : documenté |

---

## Phase 3 : Hardening

Les corrections portent sur les trois vulnérabilités de plus haute criticité (V1, V2, V3). V4 et V5, de sévérité moyenne et faible, sont documentées dans le rapport mais n'ont pas été corrigées.

### V1 : Chiffrement des préférences (`EncryptedSharedPreferences`)

`SessionManager` a été créé pour centraliser la gestion de session. Il utilise `EncryptedSharedPreferences` (Jetpack Security) : les données sont stockées dans un fichier chiffré unique (`session_enc`), protégé par une `MasterKey` AES-256-GCM gérée par l'**Android Keystore**.

```
session_enc  (EncryptedSharedPreferences)
├── user_id        → SessionManager
├── username       → SessionManager
├── is_logged_in   → SessionManager
└── db_pass_key    → KeyManager  (UUID aléatoire généré au premier lancement)
```

- Clés chiffrées : **AES-256-SIV**
- Valeurs chiffrées : **AES-256-GCM**
- Clé maître : **Android Keystore**, non extractible

### V2 : Chiffrement de la base de données (SQLCipher)

La base `lbm-db` est entièrement chiffrée avec SQLCipher (bibliothèque native `.so` fournie pour arm64-v8a, armeabi-v7a, x86 et x86_64). La clé est générée aléatoirement au premier lancement, stockée dans les `EncryptedSharedPreferences` et jamais écrite en clair. Fichiers modifiés : `MainActivity.kt` (initialisation de Room avec la fabrique SQLCipher) et `util/KeyManager.kt` (nouveau module de gestion de la clé).

```
Premier lancement
└── KeyManager.getDatabaseKey()
    └── SessionManager.getEncryptedPrefs()
        ├── Clé présente ? → retourne la clé existante
        └── Absente       → génère un UUID → le stocke chiffré → le retourne

Ouverture de la base
└── AppDatabase (lbm-db) ← clé fournie par KeyManager
    └── SQLCipher déchiffre la base en mémoire
```

### V3 : Politique de mot de passe

`PasswordPolicy` impose 8 caractères minimum, avec au moins une majuscule, une minuscule, un chiffre et un caractère spécial (`@#!%*?&`). `RegisterActivity` utilise désormais `PasswordPolicy.validate()` à la place de la simple vérification de longueur. Le verrouillage après tentatives échouées est assuré côté serveur par Firebase Authentication.

### Dépendances de sécurité

```gradle
implementation "androidx.security:security-crypto:1.1.0-alpha06"
implementation "net.zetetic:android-database-sqlcipher:4.5.4"
implementation "androidx.sqlite:sqlite-ktx:2.3.1"
```

### Vérification des corrections

Sur un émulateur, avec un build debug :

```bash
# Le fichier de préférences ne doit pas contenir de valeurs lisibles
adb shell run-as com.example.leanmasscalculator cat shared_prefs/session_enc.xml

# La base ne doit pas s'ouvrir sans clé
adb shell run-as com.example.leanmasscalculator ls databases/
adb exec-out run-as com.example.leanmasscalculator cat databases/lbm-db > lbm-db
sqlite3 lbm-db ".tables"    # doit échouer : "file is not a database"
```

---

## Couverture MASVS

Le durcissement ne couvre pas l'ensemble du MASVS. État par catégorie (MASVS v2) :

| Catégorie | Statut | Détail |
|---|---|---|
| MASVS-STORAGE | Couvert | V1 corrigée : préférences chiffrées |
| MASVS-CRYPTO | Couvert | V2 corrigée : base chiffrée, clé maître dans le Keystore ; clé de base à renforcer (voir limites) |
| MASVS-AUTH | Partiel | V3 corrigée : politique de mot de passe ; verrouillage délégué à Firebase |
| MASVS-NETWORK | Audité, non corrigé | V4 : clé API Firebase exposée |
| MASVS-CODE | Audité, non corrigé | V5 : traces de débogage |
| MASVS-PLATFORM | Hors périmètre de l'audit | |
| MASVS-RESILIENCE | Hors périmètre de l'audit | |
| MASVS-PRIVACY | Hors périmètre de l'audit | |

---

## Limites connues

- **Revue de code uniquement.** L'audit s'est appuyé sur la lecture du code source ; il n'inclut pas d'analyse dynamique (trafic réseau, instrumentation).
- **Protection au repos uniquement.** Le chiffrement protège les fichiers contre une lecture hors de l'application (appareil rooté, vol de l'appareil, extraction du stockage). Il ne protège pas contre un attaquant qui instrumente l'application à l'exécution (par exemple avec Frida) : la clé de la base peut être interceptée au moment où l'application l'utilise.
- **Données Firestore non couvertes.** Les données synchronisées dans le cloud ne sont pas concernées par le chiffrement local ; leur protection dépend des règles de sécurité Firestore.
- **Migration d'une base existante.** Passer d'une base Room non chiffrée à SQLCipher exige une stratégie de migration ou une réinstallation de l'application.
- **APK de démonstration en mode debug.** `app-debug.apk` est débogable par construction. Un build release (avec `debuggable=false`) est nécessaire pour une diffusion réelle.
- **Clé de base dérivée d'un UUID.** Environ 122 bits d'aléa : acceptable pour ce projet, mais une clé de 256 bits générée avec `SecureRandom` serait préférable.
- **Bibliothèques à migrer.** `android-database-sqlcipher` et `androidx.security:security-crypto` sont des dépendances à surveiller : vérifier leur statut de maintenance et migrer vers leurs successeurs si nécessaire.
- **Jetons Firebase.** Les jetons gérés par Firebase suivent leur propre mécanisme de stockage, distinct du chiffrement décrit ci-dessus.

### Pistes d'amélioration

- Corriger V5 : supprimer les `printStackTrace()` ou les conditionner à `BuildConfig.DEBUG`.
- Corriger V4 : restreindre la clé API Firebase (package et empreinte SHA-1) et vérifier les règles Firestore.
- Passer `allowBackup` à `false` et fournir un APK release signé, avec `debuggable=false`.
- Activer R8 (obfuscation et réduction du code).
- Générer la clé de base avec `SecureRandom` (256 bits).

---

## Installation

Ce dépôt contient l'APK de démonstration, pas le code source.

1. Télécharger [`app-debug.apk`](./app-debug.apk).
2. L'installer sur un appareil ou un émulateur Android (API 24+) :

```bash
adb install app-debug.apk
```

---

## Auteur

**EZZAIRI Wissal** - ENSA Agadir, filière Sécurité IT et Confiance Numérique
