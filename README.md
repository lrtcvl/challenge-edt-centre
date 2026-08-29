# Challenge Régional des Écoles de Tir — « La Relève au 10.9 »

Application web de gestion et de consultation du **Challenge Régional des Écoles de Tir** de la Ligue Centre-Val de Loire : 6 étapes départementales (octobre à avril) + une Finale régionale (juin), avec classement en direct par étape, palmarès général sur la saison (moyenne des 3 meilleurs scores) et classement Combiné (150/154).

Projet indépendant, construit sur le même principe que le site « Résultats en Direct » de la Ligue (Firebase Realtime Database + Firebase Authentication, hébergement statique GitHub Pages), mais avec une structure de données dédiée à un challenge multi-étapes et multi-saisons.

## Structure du projet

| Fichier | Rôle |
|---|---|
| `config.js` | **Fichier unique à configurer** (clés Firebase, nom du challenge, catégories, disciplines). Chargé par toutes les pages. |
| `ranking.js` | Logique de classement partagée (tri, places, moyenne des 3 meilleurs scores, calcul automatique du Combiné, regroupement mixte des catégories). |
| `common.css` | Style visuel commun à toutes les pages. |
| `index_admin.html` | Tableau de bord admin : création des saisons, des 6 étapes + Finale (département, lieu, dates), dépôt des PDF ISISWEB, génération et impression des QR codes. Protégé par Firebase Authentication. |
| `saisie.html` | Saisie et import CSV ISISWEB par étape (Pistolet / Carabine), corrections manuelles, statuts (DNS/DSQ/DNF/HM). Pour la Finale : bascule Qualification / Top 8. Protégé par authentification. |
| `affichage_etape.html` | Page publique d'affichage en direct des résultats d'une étape (Pistolet, Carabine, Combiné calculé automatiquement), catégories mixtes, en-têtes sticky. |
| `affichage_finale.html` | Page publique dédiée à la Finale : Qualification du matin puis Top 8 de l'après-midi. |
| `palmares.html` | Page publique : classement par étape, classement général de la saison (moyenne des 3 meilleurs scores), classement général Combiné, export PDF (impression navigateur). |
| `database.rules.json` | Règles de sécurité Firebase Realtime Database. |
| `logo.png` | Logo de la Ligue (déjà fourni). |

## Fonctionnement général

### Catégories de classement (règlement mis à jour)

**Les classements sont mixtes** (Filles et Garçons ensemble) sur 3 catégories : **Poussins**, **Benjamins**, **Minimes**.

ISISWEB continue d'exporter des catégories séparées par sexe (`PF`, `PG`, `BF`, `BG`, `MF`, `MG`) : ces codes bruts sont conservés tels quels dans la base de données (utile pour la traçabilité et la saisie), mais **tous les classements, tableaux d'affichage et palmarès regroupent automatiquement** Filles et Garçons dans la même catégorie mixte, via la fonction `mergedCategoryCode()` de `ranking.js` (ex. `PF` et `PG` → `Poussins`). Aucune action n'est nécessaire à l'import CSV : le regroupement est automatique partout (affichage étape, Finale, palmarès général et combiné).

### Modèle de données (Firebase Realtime Database)

```
seasons/
  └─ 2026-2027/                     → une saison = un dossier (permet de garder l'historique)
       ├─ config/                    → { label, active }
       └─ etapes/
            ├─ E1 (…E6)/             → une étape = un département
            │    ├─ config/          → { name, department, location, dateStart, dateEnd, order, type:"etape", scrollSpeed }
            │    │                     (dateEnd optionnel : à renseigner uniquement si l'étape se déroule sur 2 jours)
            │    ├─ shooters/        → { id: { licence, lastName, firstName, club,
            │    │                            category:"PF"|"PG"|"BF"|"BG"|"MF"|"MG" (catégorie brute ISISWEB),
            │    │                            discipline:"pistolet"|"carabine", status, score, series, mouches } }
            │    └─ pdfs/            → { pistolet, carabine, combine } fichiers PDF ISISWEB déposés
            └─ FINALE/
                 ├─ config/          → { type:"finale", … }
                 └─ shooters/        → mêmes champs + "round":"qualif"|"finale"
```

- Le **classement par étape** (Pistolet, Carabine) est alimenté par **import CSV ISISWEB** puis affiché en direct sur `affichage_etape.html`, avec possibilité de correction manuelle dans `saisie.html`.
- Le **classement Combiné 150/154** par étape est **calculé automatiquement** : un·e tireur·se qui a un score Pistolet ET un score Carabine dans la même étape obtient un score combiné = somme des deux totaux. Aucune saisie supplémentaire n'est nécessaire.
- Le **classement général de la saison** (par discipline et par catégorie mixte, moyenne des 3 meilleurs scores sur les étapes disputées) est **calculé automatiquement** par `palmares.html` à partir des résultats des 6 étapes (la Finale n'entre pas dans ce calcul). Il est imprimable en PDF via le bouton dédié (impression navigateur → « Enregistrer en PDF »), avec en en-tête le nom du challenge.
- Les **PDF officiels ISISWEB** (classement papier par étape, y compris le classement Combiné 150/154 tel que sorti d'ISISWEB) sont simplement **déposés** depuis l'admin et mis à disposition en téléchargement sur `palmares.html`. Ils sont stockés encodés dans la Realtime Database (limite indicative de 5 Mo par fichier) pour rester sur le plan gratuit Firebase (voir remarque plus bas).

### Qualification pour la Finale régionale

D'après le règlement : les **15 meilleur·e·s de chaque catégorie mixte** (Poussins, Benjamins, Minimes), ayant disputé **au moins 3 manches**, sont convié·e·s à la Finale ; les **8 meilleur·e·s** de la Qualification du matin accèdent ensuite à la Finale de l'après-midi.

- La sélection des 15 qualifié·e·s se fait **en amont**, hors application, à partir du classement général (`palmares.html`, colonne « Moy. 3 meill. », en écartant les tireurs marqués « Non classé »).
- Une fois sur place, les résultats de la **Qualification** (matin) sont importés/saisis dans `saisie.html` avec la bascule sur « Qualification » : `affichage_finale.html` classe alors automatiquement les qualifié·e·s et met en évidence les 8 meilleur·e·s de chaque catégorie (surlignage vert).
- Les résultats de la **Finale** (après-midi, top 8) sont ensuite importés/saisis avec la bascule sur « Finale — Top 8 ».

### Format du fichier CSV attendu (export ISISWEB, séparateur `;`)

```
NOM;PRENOM;LICENCE;ASSOCIATION NOM;CATÉGORIE E;EPREUVE;PLACE;SÉRIE 1;SÉRIE 2;SÉRIE 3;SÉRIE 4;NOMBRE MOUCHES
```

- `EPREUVE` doit commencer par **150** (Pistolet) ou **154** (Carabine) ; toute autre épreuve est ignorée à l'import.
- `PLACE` peut contenir `DNS`, `DSQ`, `DNF` ou `HM` pour définir le statut du tireur.
- `CATÉGORIE E` doit contenir le code brut ISISWEB (`PF`, `PG`, `BF`, `BG`, `MF` ou `MG`) : il est stocké tel quel, le regroupement mixte se fait automatiquement à l'affichage.
- `LICENCE` est **l'identifiant unique** du tireur utilisé pour le classement général sur la saison — vérifiez que cette colonne est bien exportée par ISISWEB (nom de colonne accepté : `LICENCE`, `N° LICENCE`, `NUM LICENCE` ou `N°LICENCE`). Si elle est absente, complétez le numéro de licence manuellement dans `saisie.html` avant de vous fier au classement général.

> ⚠️ Si votre export ISISWEB utilise des intitulés de colonnes légèrement différents, ouvrez `saisie.html` et adaptez les libellés recherchés dans la fonction `processCSVImport()` (variable `get(...)`).

## Démarches pour initialiser et configurer le projet

### 1. Créer le projet Firebase

1. Aller sur [console.firebase.google.com](https://console.firebase.google.com) et créer un **nouveau projet** (ex. `challenge-edt-centre`).
2. Dans **Build > Realtime Database**, créer une base de données (choisir la région `europe-west1` pour rester cohérent avec le projet d'origine), en mode **verrouillé**.
3. Aller dans **Realtime Database > Règles** et coller le contenu de `database.rules.json` fourni, puis **Publier**.
4. Dans **Build > Authentication**, activer le fournisseur **Email/Mot de passe**.
5. Dans **Authentication > Users**, créer un compte administrateur (email + mot de passe) : ce sera le compte utilisé pour se connecter à `index_admin.html` et `saisie.html`.
6. Dans **Paramètres du projet (⚙️) > Général**, section « Vos applications », cliquer sur **Ajouter une application > Web (`</>`)**, donner un nom, puis copier l'objet de configuration `firebaseConfig` proposé.

### 2. Configurer le projet

1. Ouvrir `config.js`.
2. Coller les valeurs copiées à l'étape précédente dans le bloc `firebase: { ... }`.
3. Renseigner `githubBase` avec l'URL GitHub Pages que vous obtiendrez à l'étape 3 (ex. `https://lrtcvl.github.io/challenge-edt-centre`) — cette valeur sert uniquement à générer les QR codes.
4. Vérifier/adapter si besoin `challengeName` et `ligueName`.

### 3. Déployer sur GitHub Pages

1. Déposer tous les fichiers de ce projet à la racine du dépôt GitHub `challenge-edt-centre`.
2. Dans les paramètres du dépôt, section **Pages**, activer la publication depuis la branche principale (`main`), dossier racine `/`.
3. Une fois en ligne, revenir dans `config.js` et vérifier que `githubBase` correspond bien à l'URL publiée.
4. Dans la console Firebase, **Authentication > Settings > Domaines autorisés**, ajouter le domaine `lrtcvl.github.io` (sinon la connexion admin sera refusée).

### 4. Créer la première saison

1. Ouvrir `index_admin.html`, se connecter avec le compte administrateur créé à l'étape 1.
2. Cliquer sur **+ Nouvelle saison**, renseigner par exemple l'identifiant `2026-2027` et le libellé `Saison 2026 / 2027`.
3. Les 6 étapes (E1 à E6) et la Finale sont créées automatiquement : renseignez pour chacune le département, le lieu, les dates (« Du » / « au », ce dernier optionnel si l'étape dure un seul jour) et la vitesse de défilement.

### 5. Utiliser le site pendant une étape

1. Depuis `index_admin.html`, cliquer sur **✏️ Saisie / Import CSV** de l'étape concernée.
2. Importer le CSV exporté par ISISWEB (engagements avant l'étape, puis résultats au fil de l'épreuve — l'import peut être relancé plusieurs fois, il met à jour les fiches existantes).
3. Déposer, une fois l'étape terminée, les PDF officiels ISISWEB (Pistolet, Carabine, Combiné) depuis `index_admin.html`.
4. Générer le QR code de la page d'affichage (bouton **📱 QR**), qui pointe vers une URL du type `https://lrtcvl.github.io/challenge-edt-centre/affichage_etape.html?season=2026-2027&etape=E1` (ou `affichage_finale.html` pour la Finale). Le bouton **🖨️ Imprimer l'affiche A4** génère une affiche prête à imprimer (logo, nom de l'étape, QR code en grand format, URL de secours), ou projetez directement la page d'affichage sur un écran.

### 6. Consulter les palmarès

- `palmares.html` est accessible sans connexion. Il permet de choisir la saison, de basculer entre classement par étape / classement général / classement général Combiné, de filtrer par catégorie mixte (Poussins/Benjamins/Minimes) et discipline, et de télécharger le classement affiché en PDF via le bouton **🖨️ Télécharger en PDF**.

### 7. Passer à la saison suivante

Il suffit de créer une nouvelle saison depuis `index_admin.html` (ex. `2027-2028`) : les données de la saison précédente restent consultables (sélecteur de saison dans `palmares.html` et dans l'admin).

## Remarque sur le stockage des PDF ISISWEB

Pour rester sur le plan gratuit **Spark** de Firebase (Firebase Storage nécessite désormais le plan payant Blaze), les PDF déposés sont encodés et stockés directement dans la Realtime Database, avec une limite de 5 Mo par fichier appliquée côté interface et confirmée par les règles de sécurité. Si vos PDF ISISWEB sont plus volumineux, pensez à les compresser avant dépôt, ou passez au plan Blaze et migrez vers Firebase Storage.

## Technologies utilisées

- HTML / CSS / JavaScript (vanilla, aucun framework)
- [Firebase Realtime Database](https://firebase.google.com/docs/database) et [Firebase Authentication](https://firebase.google.com/docs/auth)
- [QRCode.js](https://github.com/davidshimjs/qrcodejs)
- Hébergement statique compatible GitHub Pages

## Points restés en simplification (à faire évoluer si besoin)

- La sélection des 15 qualifié·e·s à la Finale se fait manuellement par l'admin à partir du classement général (pas de verrouillage automatique dans l'app).
- La Finale suppose que vous importez/saisissez séparément les résultats de Qualification et de Finale (bascule en haut de `saisie.html`).
- Le classement général et le classement Combiné général sont recalculés à la volée à chaque affichage de `palmares.html` (pas de valeur mise en cache) : au-delà de quelques centaines de tireurs cumulés sur 6 étapes, cela reste largement instantané pour ce volume de compétition régionale.