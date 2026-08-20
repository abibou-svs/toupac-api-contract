# TOUPAC — App CONTRÔLEUR — Brief technique dev RN

> **Pour** : dev mobile React Native
> **Objectif** : livrer l'app CONTRÔLEUR (mode offline) pour le **19 septembre 2026**
> **Références complètes** : `TOUPAC-VOYAGE-MVP-SPEC.md` (spec détaillée) + `TOUPAC-VOYAGE-OPENAPI.yaml` (contrat API, livré fin de semaine)
> **Version** : 1.1 — 19 août 2026
> **Changement v1.0 → v1.1** : refonte backend (mapping natif Fleetbase pour voyage/escales/colis). **Aucun impact sur ton côté** : le contrat API reste identique. Point de vigilance : le format QR est **JWT signé RS256** (au lieu de UUID+HMAC) — voir stack ajoutée §2 + question 1 §10 résolue.

---

## 1. Contexte en 30 secondes

L'app CONTRÔLEUR est utilisée par un agent à bord d'un bus interurbain (Bamako↔Abidjan, etc.). Il vérifie les billets passagers, les colis, peut vendre des billets aux walk-in en cash, gère les cas spéciaux et les incidents, puis clôture le voyage à l'arrivée.

**Contrainte n°1 : la connexion mobile est instable ou absente pendant le trajet.** L'app doit fonctionner à 100% offline entre le départ et l'arrivée, avec sync automatique dès qu'un réseau est détecté.

---

## 2. Stack recommandée

Tu es libre de tes choix, mais voici la reco :

| Domaine | Reco | Alternative |
|---|---|---|
| Language | TypeScript strict (`"strict": true` dans tsconfig, pas de `any` non justifié) | — |
| Storage local | **WatermelonDB** (ORM offline-first, sync friendly) | `react-native-quick-sqlite` + wrapper maison |
| Chiffrement DB | **SQLCipher** intégré via `op-sqlite` | — |
| Secure storage tokens | `react-native-keychain` | — |
| État global | Zustand ou Jotai | Redux Toolkit si tu préfères |
| HTTP client | `ky` ou `axios` avec interceptor retry + Idempotency-Key auto | `fetch` natif |
| Détection réseau | `@react-native-community/netinfo` | — |
| Scanner QR | `react-native-vision-camera` + `vision-camera-code-scanner` | `expo-camera` |
| Vérif JWT billet (v1.1) | `react-native-jwt-io` — RS256, clé publique embedded dans le bundle | Implémentation maison JWT (à éviter) |
| Upload photos | Direct S3 via URLs présignées (jamais via serveur) | — |
| Navigation | React Navigation v7 | — |
| Formulaires | React Hook Form + Zod | Formik + Yup |

**Non recommandé** : AsyncStorage plain pour les tokens (**JAMAIS**), Redux si tu débutes avec (Zustand fait 90% du taf en 10% du boilerplate).

---

## 3. Les 5 écrans à livrer

Le Figma contient les designs. Voici la liste complète des écrans + le comportement attendu.

### Écran 1 — Connexion Contrôleur
- **Figma** : `Connexion - Contrôleur`
- **API** : `POST /auth/login` → `{access_token, refresh_token, controller: {...}}`
- **Tokens** : `access_token` = 30 min, `refresh_token` = **30 jours** (pour supporter longues périodes offline)
- **Stockage** : Keychain iOS / EncryptedSharedPreferences Android — **jamais** AsyncStorage plain
- **Offline** : si `refresh_token` valide en cache → écran PIN direct (bypass login serveur)

### Écran 2 — Dashboard / Voyages du jour
- **Figma** : `Accueil - controller`, `Notifications`, `notification-detail-depart`, `notification-detail-arret`
- **API** : `GET /trips?date=today&assigned_to_me=true`
- **Data** : 4 compteurs top (Voyages / Passagers attendus / Colis / Incidents), liste voyages du jour, activité récente, cloche notifs
- **Offline** : dernière liste cachée en local, badge "Dernière synchro il y a Xh Ym", compteur global "N actions en attente" en header
- **Notifications** : **mockées V1** (compteur local uniquement, pas de push serveur — décision produit)

### Écran 3 — Détail voyage + Préparation du contrôle
- **Figma** : `detail-voyage-controleur`, `CTRL — Préparation du contrôle`
- **API** :
  - `GET /trips/{id}` → détail voyage
  - `GET /trips/{id}/manifest` → **payload complet** (passagers + colis + politique bagages + plan sièges + escales) — c'est l'appel critique offline
  - `POST /trips/{id}/control/open` → crée session
- **Checklist 7 items** (manifeste téléchargé, bordereau colis, données offline OK, appareil autorisé, scanner OK, batterie OK, dernière sync)
- **Comportement critique** : cet écran **exige un réseau** pour télécharger. Une fois le manifeste stocké localement, tout le reste (écrans 4-5) fonctionne offline.

### Écran 4 — Contrôles (cœur de l'app, multi-onglets)
3 onglets + une dizaine de sous-écrans. Détaillé ci-dessous en §4.

### Écran 5 — Clôture du contrôle
- **Figma** : `Chauff - Accueil` (mal labellisé, c'est le récap embarquement), `CTRL — Clôture du contrôle`, `manifeste-de-cloture`
- **API** : `POST /trips/{id}/control/close` avec récap `{passengers, parcels, cash, discrepancies}`
- **Section ajoutée V1 non-Figma — Réconciliation cash** :
  - Bloc "Ventes à bord" : liste ventes onboard avec passager + montant + siège
  - Total collecté (ex 45 000 XOF)
  - Champ "Montant remis à l'agence" (saisie manuelle)
  - Différence calculée auto
- **Contrainte** : **la clôture EXIGE d'être en ligne + queue vide**. Bouton "Clôturer" grisé sinon.

### Overlay permanent — Centre de synchronisation
- **Figma** : `CTRL — Centre de synchronisation`
- Accessible depuis n'importe où via badge "N en attente" en header
- Progress bar globale + statut par catégorie (passagers/colis/anomalies/incidents/ventes) + dernières actions locales + bouton retry par action en échec

---

## 4. Écran 4 en détail (le plus lourd)

### Onglet Passagers
- **Figma** : `Contrôles` (onglet Passagers)
- Recherche par nom/téléphone/siège, chips filtres (Tous/Embarqués/En attente), compteurs cliquables, liste passagers
- Tap passager → écran `fiche-passager` (détail avec DDN, CNI, tél, siège, heure, trajet)
- Actions par passager : "Valider embarquement" (event `reservation_board`) / "Cas spécial" (écran cas spécial) / "Refuser" (motifs)
- **API** : lecture = snapshot local. Actions = queue offline.

### Onglet Colis
- **Figma** : `Contrôles - Colis`
- Chips filtres, liste colis avec référence + destinataire + poids + destination + statut
- Tap colis → `Colis Detail` (photos, notes, validation)
- Actions : "Valider" (event `parcel_verify`), "Ajouter photo" (upload différé S3), "Signaler anomalie"

### Onglet Scanner QR
- **Figma** : `Contrôles` (Scanner QR), `Scan embarquement`, `Scan Success`, `Scan Warning`, `Accueil - v2`, `Scan Colis`
- Overlay caméra live + décodage QR
- **4 cas de scan** :
  - ✅ Billet valide → écran `Scan Success` (bloc vert) → bouton "Valider l'embarquement" → event `reservation_board`
  - ⚠️ Déjà embarqué → écran `Scan Warning` (bloc orange, comparaison horaires 1er vs actuel) → 2 boutons (Retour / Cas particulier) + **création automatique en background** d'une `Anomaly` sévérité `moderate`
  - ❌ Billet expiré / autre voyage → écran "Billet expiré" → bouton Refuser (event `reservation_refuse`) ou Cas particulier
  - ❓ QR invalide → écran "QR Code Invalide" → boutons Réessayer / Saisie manuelle
- **Bouton flottant "Saisie manuelle"** toujours accessible (fallback QR abîmé)

### Sous-écran Saisie manuelle
- **Figma** : `Saisie manuelle - Formulaire`, `Saisie manuelle - Billet trouvé`, `Saisie manuelle - Aucun résultat`, `Saisie manuelle - Plusieurs résultats`
- Recherche par référence billet OU par nom/tél/siège
- **Comportement** : recherche dans le snapshot local d'abord (offline-friendly), puis API `GET /reservations?trip_id=X&search=Y` si réseau et rien trouvé
- 3 UI selon résultat : 1 hit / 0 hit / N hits

### Sous-écran Cas Spécial
- **Figma** : `cas-special-controleur`
- 4 types exclusifs : Enfant <4 ans (gratuit) / Accompagnant / Cas médical / Autre
- Commentaire (200 chars max) + pièces jointes photo/PDF
- Section conditionnelle "Motifs de refus" (chips)
- Boutons "Autoriser" (event `special_case` avec `authorized: true`) / "Refuser" (event `refuse` avec motif)

### Sous-écran Vente onboard (walk-in cash) ⚠️ **PAS DE FIGMA**
- Écran à concevoir en collaboration avec le designer (ou en solo si urgence). Structure minimale :
  - Champ Nom passager (obligatoire) + Téléphone (obligatoire)
  - Sélecteur siège : dropdown des sièges marqués "libre" dans le snapshot local
  - Sélecteur destination : dropdown des `TripStop` restantes
  - Prix affiché (calculé auto selon `Route.default_price_xof`)
  - Bouton "Encaisser cash et valider"
- **Event queue** : `onboard_sale` avec `{passenger, seat, stop, price_xof, payment_method: cash, client_uuid}`
- **Le serveur peut rejeter à la sync** si le siège a été pris entre-temps par une vente app Client → voir §6

### Sous-écran Anomalies
- **Figma** : `CTRL — Liste des anomalies`, `CTRL — Détail anomalie`
- Liste avec chips filtres + compteurs (Total, En attente, Critiques, Résolues)
- Détail : bloc sévérité + statut + description + historique des scans (traceability anti-fraude) + boutons "Signaler un incident" / "Refuser l'embarquement"
- **Note importante** : la plupart des anomalies sont créées **automatiquement par le backend** (double-scan, seat conflict). Le contrôleur les consulte et les résout.

### Sous-écran Incidents
- **Figma** : `incident-nouveau`, `incident-detail`
- Nouveau : titre, type (Bagages/Comportement/Sécurité/Panne/Autre), sévérité (Faible/Modéré/Critique), description, photos, GPS auto-détecté, toggle "Notifier le dispatcher"
- Détail : timeline (Signalé → Notifié → En cours → Résolu), photos, GPS résolu en adresse
- **GPS** : récupéré depuis le device de l'app (pas depuis le serveur), stocké sur l'incident. Aucun conflit avec la position GPS du chauffeur.

---

## 5. Comportement offline — spécification technique

### 5.1 Queue d'events locale
Table SQLite locale `control_events_queue` :
```
client_uuid       TEXT PRIMARY KEY       -- UUID v4 généré côté client
session_id        TEXT NOT NULL
event_type        TEXT NOT NULL          -- reservation_board, parcel_verify, onboard_sale, etc.
target_type       TEXT NOT NULL          -- reservation, entity (parcel FLB natif), anomaly, incident
target_uuid       TEXT                   -- UUID cible côté serveur (FLB natif = uuid CHAR(36)) ; null pour create (onboard_sale par ex)
payload_json      TEXT NOT NULL
created_at_local  DATETIME NOT NULL
status            TEXT NOT NULL          -- pending / syncing / sent / failed
retry_count       INTEGER DEFAULT 0
last_error        TEXT
gps_lat           REAL
gps_lng           REAL
```

### 5.2 Upload en batch
- Endpoint : `POST /control-events/batch` (max **100 events** par requête)
- Header obligatoire : `Idempotency-Key: <uuid-de-la-requête-batch>`
- Chaque event a son propre `client_uuid` qui garantit l'idempotence côté serveur
- Le serveur traite **transactionnellement par event** et retourne un verdict individuel :
  ```json
  {
    "results": [
      { "client_uuid": "abc123", "status": "accepted", "server_id": "res_xxxxx" },
      { "client_uuid": "def456", "status": "rejected", "conflict_reason": "seat_taken_online" }
    ]
  }
  ```

### 5.3 Retry policy
- Automatique en background dès qu'un réseau est détecté (`NetInfo`)
- Retry exponentiel sur échec réseau : 1s, 2s, 4s, 8s, 16s, 32s → puis pause jusqu'à action user
- **Arrêt immédiat sur échec métier** (rejected par serveur) → notification user + affichage dans Centre de sync

### 5.4 Upload photos (différencié des events)
- Endpoint : `POST /uploads/presign` → retourne `{upload_url, public_url}`
- Upload direct S3 en PUT
- Une fois uploadé, le `public_url` est référencé dans le payload de l'event correspondant
- Queue photos séparée de la queue events (photos = potentiellement gros, traitement différent)

### 5.5 Snapshot manifeste local
- Téléchargé une fois via `GET /trips/{id}/manifest` à l'ouverture du contrôle
- Stocké en SQLite avec `snapshot_taken_at`
- Le contrôleur **peut re-synchroniser** (bouton "Synchroniser les données") mais cela **n'écrase pas** les events déjà en queue locale
- Après clôture + 7 jours grâce → purge auto

---

## 6. Conflit siège — le seul cas complexe

Scénario : le contrôleur vend le siège A14 en offline. Entre-temps, un client a acheté A14 via app Client (l'app client V1 n'existe pas encore, mais le protocole est prêt pour V2).

**Comportement attendu** :
1. Contrôleur vend siège A14 offline → event `onboard_sale` en queue
2. Sync → serveur rejette l'event : `{status: "rejected", conflict_reason: "seat_taken_online"}`
3. Serveur crée auto une `Anomaly` sévérité `critical`
4. App CONTRÔLEUR reçoit le verdict et **affiche une notification post-sync** : "Conflit détecté — passager Y à bord sans siège valide"
5. UX de résolution — 2 boutons :
   - **"Réattribuer un siège libre"** → liste des sièges dispo actuels → nouvelle vente
   - **"Rembourser cash"** → enregistre `CashEntry` négative → anomalie résolue

Pour V1 (pas d'app Client), ce scénario est théorique mais **on l'implémente quand même** parce qu'il fait partie de la démo au client (scénario 3 en §7 ci-dessous).

---

## 7. Critères d'acceptation démo — 3 scénarios à jouer en live

### Scénario 1 — Voyage nominal
1. Contrôleur se connecte
2. Voit son voyage du jour Bamako→Abidjan
3. Ouvre le voyage, complète la checklist préparation, télécharge manifeste
4. **Passe le device en avion** (démo offline)
5. Scanne 3 billets valides → 3 passagers embarqués
6. Fait 1 cas spécial (enfant <4 ans, autorisé)
7. Vend 1 billet onboard cash à un walk-in
8. Valide 2 colis
9. **Repasse en ligne** → tous les events se synchronisent en <30s
10. Clôture le voyage, réconciliation cash affichée

### Scénario 2 — Gestion anomalie
1. Contrôleur scanne un billet valide (embarquement OK)
2. **Rescanne le même billet** → écran "Déjà embarqué"
3. Anomalie créée automatiquement (visible dans Liste anomalies)
4. Contrôleur consulte le détail anomalie, marque "signaler incident" → incident créé

### Scénario 3 — Conflit siège onboard vs app
1. Contrôleur télécharge manifeste, part en mode offline
2. Un test manuel côté console : marquer siège A14 comme vendu à un autre passager
3. Contrôleur vend siège A14 onboard offline
4. Repasse en ligne → sync
5. Notification "Conflit détecté sur siège A14"
6. Contrôleur choisit "Réattribuer un siège libre" → nouvelle vente sur siège A15

---

## 8. Hors scope V1 (ne code PAS ces features)

- App CLIENT (billetterie + envoi colis + upload identité) → V2
- App CHAUFFEUR → V2
- Vente onboard **mobile money** (Wave, OM, Free Money) → **CASH ONLY V1**
- Push notifications réelles (FCM/ExpoPush) → **mockées V1**
- Auth biométrique locale → **Code PIN suffit V1**
- Multi-langue (EN/WO) → **FR only V1**
- Édition politique bagages en app → **lecture seule V1**

---

## 9. Contraintes techniques transverses

### 9.1 Sécurité
- Tokens dans Keychain / EncryptedSharedPreferences uniquement
- SQLCipher pour la DB locale (surtout à cause des CNI passagers en cache)
- HTTPS strict, pas de bypass cert en release build
- Logs : ne jamais logger tokens ni CNI

### 9.2 Performance
- Manifeste peut contenir 50+ passagers + 30+ colis + photos → **< 500 KB total** (le backend compresse et fournit des URLs photos, pas les blobs)
- Recherche locale doit être instant (< 100ms) même sur 100 passagers
- Scan QR : décodage < 500ms

### 9.3 Résilience
- Si sync échoue plusieurs fois → **jamais bloquer l'UX**. Toujours possible de continuer à travailler offline.
- Si manifeste corrompu localement → bouton "Retélécharger le manifeste" (safeguard debug)

---

## 10. Points à clarifier avec le lead backend avant de coder

Pose-lui ces questions cette semaine avant J3 (livraison OpenAPI) :

1. ~~**Structure exacte du QR**~~ **[Résolu v1.1]** : **JWT signé RS256**. Payload : `{iss: "toupac", sub: "res_xxxxxxxxxx", ord: "order_xxxxxxxxxx", seat: "A14", ref: "BIL-2025-4521", iat, exp}`. Clé publique RS256 embedded dans l'app (fournie via constante ou bundle asset à la compilation — chemin exact livré J3). Décodage 100% offline : vérifie signature + `exp` claim + `ord` correspond au voyage en cours.
2. **Précision GPS incident** : format `{lat, lng}` seul ou avec `accuracy_meters` + `altitude` ?
3. **Format des dates** : ISO 8601 UTC partout, tu fais la conversion timezone côté app ?
4. **Rate limiting API** : combien de requêtes/min max ? Impacte le batching queue.
5. **Handling des refresh tokens expirés** : le serveur retourne 401 avec quel body précis ? Comment déclencher un logout complet et une reconnexion clean ?

---

## 11. Livraisons attendues côté toi

| Milestone | Date cible | Livrable |
|---|---|---|
| Setup projet | 26 août | Repo RN créé, dépendances installées, écran login + Dashboard connectés à mocks Postman |
| Écrans 3-4 fonctionnels | 5 sept | Écrans détail voyage + contrôles + saisie manuelle + cas spéciaux fonctionnent contre backend réel |
| Offline complet | 12 sept | Queue events + retry + sync = OK, tous les scénarios offline passent |
| Hardening + démo | 18 sept | Tests end-to-end verts, dry-run avec le lead |
| **Jour J** | **19 sept** | Démo client |

---

## 12. Points de synchro proposés

- **Slack quotidien matin** : 3 lignes async (fait / en cours / bloqué)
- **Design review lundi** (30 min live) : blockers API, évolutions design
- **Demo intermédiaire vendredi** (30 min) : ce qui marche, ce qui reste

Toute évolution API = message Slack au lead + mise à jour de l'OpenAPI en source unique.

---

**Fin du brief — Version 1.1**

Questions techniques → contacte le lead backend directement. Questions produit → escalader au lead pour arbitrage client.
