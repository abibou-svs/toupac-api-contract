# TOUPAC — Extension `toupac-voyage` — MVP Spec figée

> **Ce que c'est** : spécification figée du MVP de l'extension `toupac-voyage`, focalisée sur l'app mobile CONTRÔLEUR à livrer le **19 septembre 2026**.
> **Ce que ce n'est pas** : spec fonctionnelle complète du module Voyage TOUPAC. Beaucoup de features sont volontairement reportées en V2 (voir §12).
> **Version** : 1.1 — 19 août 2026
> **Statut** : brouillon, en attente de validation client
> **Changement v1.0 → v1.1** : passage à l'approche C hybride (mapping natif Fleetbase pour voyage/escales/colis). Voir §1.2 et `TOUPAC-VOYAGE-CHANGELOG.md`. Contrat API inchangé côté app mobile.

---

## 1. Contexte

### 1.1 Le module en une phrase
`toupac-voyage` est l'extension Fleetbase qui ajoute la **billetterie interurbaine passagers + colis** au fork TOUPAC, avec un focus V1 sur les workflows du CONTRÔLEUR à bord.

### 1.2 Positionnement vs Fleetbase natif (approche C hybride — v1.1)

**Massivement réutilisé** (aucune duplication, on utilise les tables natives directement) :
- **`Order`** = le voyage physique (avec `order_config = passenger_trip` custom)
- **`Payload`** = conteneur du voyage (contient escales + colis)
- **`Waypoint`** = les escales du voyage (avec ordre séquentiel, time windows, POD)
- **`Entity`** = les colis (avec `type = parcel`, qr_code natif, tracking number natif, POD photos)
- **`OrderConfig`** = définit le workflow custom `passenger_trip` (activity flow BOARDING → IN_TRANSIT → AT_STOP → ARRIVING → CLOSING)
- **`Vehicle`, `Driver`, `Place`, `User`, `Contact`, `File`, `Position` (GPS chauffeur), `Proof` (POD), `TrackingNumber`, `Route`, IAM** — tous natifs

**Non réutilisé** : `Payments` natif (Stripe uniquement, inutilisable en Afrique de l'Ouest).

**Custom (10 tables `toupac-voyage`)** :
- `passengers` — fiche passager avec CNI chiffrée (RGPD)
- `reservations` — billet passager avec siège numéroté + statut embarquement, lié à `order_uuid`
- `luggage_policies` — configuration bagages par voyage
- `seat_maps` — plan sièges par véhicule
- `controllers` — extension User FLB pour rôle contrôleur
- `control_sessions` — session contrôleur-voyage, liée à `order_uuid`
- `control_events` — queue serveur events offline (idempotence via `client_uuid`)
- `anomalies` — événements anormaux (double-scan, seat_conflict…)
- `incidents` — événements terrain (bagages, comportement, panne…)
- `cash_entries` — encaissements cash (vente onboard, remboursement)

Détail entités et relations : voir `TOUPAC-VOYAGE-ERD.md` (livraison J2).

### 1.3 Contraintes qui pèsent sur toute décision
| Contrainte | Impact concret sur cette spec |
|---|---|
| Souveraineté data Sénégal / Afrique de l'Ouest | Pas de FCM Google en prod (mock V1, décision reportée). Stockage photos S3-compatible sur cloud africain. |
| Multi-tenant Fleetbase | Toutes les tables ont `company_id`, tous les endpoints scopés par token. |
| Offline-first mobile | Tous les endpoints d'écriture idempotents (header `Idempotency-Key`). Queue d'events côté client. |
| Temps de réponse < 2s | Endpoint `GET /trips/{id}/manifest` doit être <500ms pour permettre le pré-départ rapide. |
| RGPD-like sur données perso | Fiche passager avec CNI = chiffrement au repos + audit d'accès. |
| AGPL-3.0 (phase dev) | Extension isolée dans `packages/toupac-voyage/`, aucun patch du core. |

---

## 2. Scope figé du MVP 19 septembre

### 2.1 In scope (livraison ferme)

**Côté app mobile CONTRÔLEUR** — 5 écrans macro :

1. **Connexion** — login email/password + fallback biométrie/PIN offline
2. **Dashboard** — liste des voyages du jour + compteurs
3. **Détail voyage + Préparation du contrôle** — checklist pré-départ + download manifeste
4. **Contrôles** — 3 onglets (Passagers / Colis / Scanner QR) + sous-écrans (fiche passager, cas spéciaux, vente onboard cash, anomalies, incidents)
5. **Clôture du contrôle** — récap embarquement, réconciliation cash, sync obligatoire

**Overlay permanent** :
- Centre de synchronisation (queue offline, retry, quota, sync state)

**Côté backoffice Fleetbase** — minimum viable pour supporter la démo :
- CRUD `Trip`, `Route`, `Passenger`, `Reservation`, `Parcel`, `LuggagePolicy` en console Ember
- Console d'affectation contrôleur à un voyage
- Console de consultation des sessions de contrôle et anomalies
- **Note** : les CRUD peuvent être minimalistes (formulaires bruts sans UX travaillée) — priorité à l'API et à l'app mobile

**Backend `toupac-voyage`** :
- **10 modèles Eloquent custom** + migrations (au lieu de 13 en v1.0 — voir `TOUPAC-VOYAGE-ERD.md` v1.1)
- **1 seeder `OrderConfig`** créant le workflow custom `passenger_trip`
- ~19 endpoints REST (détail dans `TOUPAC-VOYAGE-OPENAPI.yaml`, livraison J3)
- Seeder démo : 1 `Order` `type=passenger_trip` Bamako→Abidjan avec 32 `Reservation` (passagers) + 13 `Entity` `type=parcel` (colis)
- Tests feature Laravel sur les endpoints critiques (auth, manifest, sync batch, close) + tests multi-tenant isolation

### 2.2 Out of scope explicite V1 (reporté en V2)

| Feature | Reporté car | Backlog |
|---|---|---|
| App mobile CLIENT (upload identité, achat billet, envoi colis) | Charge de dev × 2 pour équipier RN, irréaliste 22 jours ouvrés | V2 |
| App mobile CHAUFFEUR (adapté depuis Navigator Fleetbase) | Non demandé dans le jalon 19 sept | V2 |
| App mobile AGENCE / vente comptoir | Non demandé dans le jalon 19 sept | V2 |
| Vente onboard **mobile money** (Wave, OM, Free Money, MTN MoMo) | Nécessite réseau au moment de la vente → casse offline-first + intégration provider non prête | V2 |
| Push notifications réelles (FCM / ExpoPush / autre) | Décision provider reportée (contrainte souveraineté vs simplicité). Notifs = **mockées** en V1 (compteur local, pas de push serveur) | V2 |
| Auth biométrique locale (empreinte / face) | Code PIN suffit en V1. Biométrie = amélioration UX V2 | V2 |
| Édition de la politique bagages en console | V1 = **lecture seule** côté contrôleur, config par seed / SQL direct | V2 (formulaire console) |
| Moteur de règles bagages avancé (VIP, promotions, zones) | Overkill V1. Structure simple `{free_count, extra_price, weight_limit, over_weight_price}` suffit | V2 |
| Reporting BI avancé (dashboards, exports PDF/Excel) | Fleetbase natif suffit V1 | V2 |
| Multi-langue de l'app CONTRÔLEUR | FR only V1 (marché SN/UEMOA francophone majoritairement) | V2 (EN, WO) |
| Intégration OSRM auto-hébergé | Navigation côté chauffeur, hors périmètre app contrôleur V1 | V2 |
| WhatsApp Business API | Notifs mockées V1 | V2 |

### 2.3 Ce qui reste ambigu — hypothèses par défaut prises en attendant validation

| # | Ambiguïté | Hypothèse prise pour avancer | Impact si erreur |
|---|---|---|---|
| A | Storage photos S3 (Wasabi / Scaleway / Raxio) | S3 local Fleetbase natif en dev, décision reportée pour la prod | Faible — abstraction Laravel Storage |
| B | Contrôleur a-t-il accès à la console web ? | **Non** — mobile only en V1. Accès console = rôle Fleet Supervisor si besoin de consultation | Moyen — création rôle IAM à ajuster |
| C | Manifeste re-téléchargeable après ouverture session ? | **Oui** — bouton "Re-synchroniser" tant que session pas close, MAIS ne réécrase pas les events déjà enregistrés localement | Faible |
| D | Que se passe-t-il si un ajout de dernière minute est fait en console pendant contrôleur offline ? | **Perdu pour ce voyage** — le contrôleur ne verra pas l'ajout (snapshot figé à `T`). Le passager doit être traité en vente onboard | Moyen — à documenter dans procédures agences |
| E | "Non ctrl" (passagers/colis non contrôlés) bloque-t-il la clôture ? | **Non bloquant** — warning affiché, le contrôleur peut fermer avec justification | Faible |
| F | Un billet peut-il couvrir plusieurs sièges (groupe) ? | **Non V1** — 1 billet = 1 siège = 1 passager. Groupe = N billets liés par `group_id` (backlog V2) | Moyen |
| G | Quel comportement scan billet quand il appartient à un autre voyage (jour ou destination différente) ? | Écran "Billet expiré / autre voyage" → refus automatique + création anomalie critique | Faible |
| H | Anti-double-scan : action auto ou choix contrôleur ? | 2ᵉ scan → écran "Déjà embarqué" affiché + création `Anomaly` sévérité `moderate` (auto) + choix contrôleur : ignorer / signaler fraude | Faible |
| I | Combien de sièges buffer réservés à la vente onboard ? | **0 par défaut** — décision C1 (virtuel > physique) rend le buffer inutile. Contrôleur peut vendre sur tout siège marqué libre au moment du snapshot | Décidé (§3.2) |
| J | Le contrôleur peut-il modifier un `LuggagePolicy` en cours de voyage ? | **Non** — politique figée au départ. Anomalies sur cas particuliers uniquement | Faible |

**Ces hypothèses seront revalidées avec le client** avant la fin du sprint 1 (livrable envoyé asynchrone).

---

## 3. Décisions actées (rappel synthétique)

Issues des échanges des 18-19 août 2026 :

| Réf | Sujet | Décision |
|---|---|---|
| A | App Client dans scope 19 sept ? | **Non**, stubbée avec seeder de passagers de test |
| B | Vente onboard : moyens de paiement | **Cash-only V1** + écran réconciliation cash à la clôture |
| C | Conflit siège virtuel vs physique | **Virtuel > physique** : vente onboard rejetée à la sync si siège pris entre-temps → anomalie critique + UX de résolution |
| D | Modèle politique bagages | **Simple V1** : `{free_luggage_count, extra_luggage_price, weight_limit_kg, over_weight_price_per_kg}` |
| E | Parcels de non-voyageurs | **Comptoir agence uniquement V1** (via console) — l'app Client V2 ajoutera la voie mobile |

### 3.1 Cas C — détail du protocole conflit

**Séquence** :
1. Contrôleur télécharge manifeste à `T0` (avant départ). Snapshot local : siège A14 = libre.
2. Contrôleur perd le réseau à `T0 + 5min`.
3. Passager X achète siège A14 via app Client à `T0 + 10min` (serveur enregistre).
4. Contrôleur vend siège A14 au walk-in Y à `T0 + 30min` (offline, event en queue avec `client_uuid=abc123`).
5. Contrôleur retrouve du réseau à `T0 + 90min`. Queue s'upload.
6. Serveur reçoit `event { type: onboard_sale, seat: A14, ... }`. Vérifie transactionnellement → siège pris par X → **rejette** avec `status: rejected, conflict_reason: seat_taken_online`.
7. Serveur crée automatiquement une `Anomaly { severity: critical, type: seat_conflict }`.
8. App CONTRÔLEUR affiche notification post-sync : "Conflit détecté — passager Y à bord sans siège valide".
9. UX de résolution : 2 boutons — (a) "Réattribuer un siège libre" (liste sièges dispo actuels) → génère nouvelle reservation, (b) "Rembourser cash" → enregistre `CashEntry` négative, marque anomalie résolue.

### 3.2 Cas I — buffer sièges onboard

Décision : **0 buffer par défaut**. Justification :
- Décision C rend le buffer inutile en tant que mécanisme anti-conflit (le serveur tranche).
- Le buffer aurait été utile pour **garantir** la vente onboard sans risque de conflit, mais on préfère laisser le choix stratégique à la V2.
- **Backlog V2** : si le taux de conflits dépasse un seuil observable en prod (à définir), ajouter un buffer configurable par route.

---

## 4. Personas & rôles

### 4.1 Persona principal V1 : le CONTRÔLEUR à bord
- **Profil** : agent employé par la compagnie de transport (ex : Toupac Transport), à bord du bus pendant le trajet
- **Terrain** : bus interurbain (Bamako↔Abidjan, Dakar↔Kaolack…) — connexion mobile souvent instable ou absente entre les grandes villes
- **Device** : smartphone Android moyen de gamme (référence Figma : Samsung A54)
- **Compétences** : lecture/écriture FR courant, familier avec les smartphones, non technicien
- **Objectifs métier** :
  1. Vérifier tous les billets à l'embarquement (scan QR ou saisie manuelle)
  2. Vérifier tous les colis à l'embarquement (scan + validation poids/état)
  3. Vendre des billets aux walk-in en cours de route (cash)
  4. Gérer les cas spéciaux (enfants gratuits, accompagnants, médicaux)
  5. Remonter les anomalies et incidents
  6. Clôturer le voyage avec réconciliation cash + rapport

### 4.2 Nouveau rôle IAM à créer : `Controller` (v1.1)
Non présent dans les 27 rôles Fleetbase natifs. À créer dans `toupac-voyage` avec les policies :
- `controller:orders:read` (voyages FLB `type=passenger_trip` assignés uniquement)
- `controller:orders:manifest:read`
- `controller:control-sessions:*` (CRUD sur ses propres sessions)
- `controller:reservations:board`
- `controller:reservations:refuse`
- `controller:reservations:special-case`
- `controller:reservations:create` (vente onboard uniquement)
- `controller:entities:verify` (colis natifs FLB, met à jour `meta.status_toupac`)
- `controller:entities:refuse`
- `controller:anomalies:create/resolve`
- `controller:incidents:create`
- `controller:passengers:read` (sans CNI)
- `controller:passengers:read-id-card` (**avec** CNI, audit obligatoire)
- `controller:orders:activity-transition` (synchronise activity flow FLB)

Scope : le contrôleur ne voit que ses propres voyages assignés + les voyages de son agence de rattachement (à confirmer §2.3.B).

### 4.3 Personas hors scope V1 (mentionnés pour cadrage)
- **Client** : achète billet et envoie colis via app CLIENT (V2)
- **Chauffeur** : conduit et fait remonter position GPS (V2, adaptation Navigator App)
- **Agent d'agence** : vend billets et prend en dépôt les colis via console web (V1 minimal, formulaires bruts)
- **Dispatcher / Superviseur** : orchestre les voyages, reçoit les incidents (V1 via console Fleetbase existante)

---

## 5. Détail des 5 écrans MVP

Chaque écran est décrit selon 5 axes : Data affichée, Appels API, Comportement offline, Interactions, Sources Figma.

### 5.1 Écran 1 — Connexion Contrôleur

**Sources Figma** : `Connexion - Contrôleur`

**Data affichée** : logo Toupac Transport, sous-titre "Application Contrôleur", inputs email + password, checkbox "Se souvenir de moi", lien "Mot de passe oublié", numéro support "+223 20 22 00 00".

**Appels API** :
- `POST /auth/login` avec `{email, password}` → retour `{access_token, refresh_token, controller: {id, name, agency}}`

**Comportement offline** :
- Si device a un `refresh_token` valide en Keychain et l'utilisateur a déjà activé le PIN local → afficher directement l'écran PIN (bypass login serveur)
- Si aucun `refresh_token` valide → login serveur obligatoire (bloquant sans réseau au premier lancement)
- `access_token` durée = 30 min ; `refresh_token` durée = **30 jours** (pour supporter les longues périodes offline)

**Interactions** :
- Tap "Se connecter" → API call
- Tap "Mot de passe oublié" → écran hors scope V1 (afficher toast "Contactez votre agence")

**Critères d'acceptation** :
- [ ] Login serveur OK avec compte contrôleur valide
- [ ] Message d'erreur clair si credentials invalides
- [ ] Tokens stockés en Keychain iOS / EncryptedSharedPreferences Android (**jamais** AsyncStorage plain)
- [ ] Écran PIN au relancement de l'app si session déjà établie

---

### 5.2 Écran 2 — Dashboard / Voyages du jour

**Sources Figma** : `Accueil - controller`, `Notifications`, `notification-detail-depart`, `notification-detail-arret`

**Data affichée** :
- Salutation "Bonjour, {controller.name}" + date du jour
- **4 compteurs top** : Voyages à contrôler (3), Passagers attendus (96), Colis à vérifier (12), Incidents remontés (2)
- **Liste "Voyages du jour"** : bloc par voyage avec route (Bamako → Abidjan), heure départ (07:30), nombre passagers (32), statut (En cours / Planifié / Terminé)
- **Bloc "Activité récente"** : 3-4 dernières actions (Scan ticket validé, Incident signalé, Colis vérifié, Embarquement validé)
- **Cloche notifications** en haut à droite avec badge non lues

**Appels API** :
- `GET /trips?date=today&assigned_to_me=true` → liste voyages du jour du contrôleur
- `GET /notifications?unread=true&limit=8` (si scope V1 — sinon mock pur)

**Comportement offline** :
- Si offline : afficher dernière version cachée en SQLite local, badge "Dernière synchro il y a Xh Ym"
- Compteur "N actions en attente" affiché en permanence dans le header si queue non vide

**Interactions** :
- Tap sur un voyage → écran 3 (Détail voyage)
- Tap cloche notifications → liste notifications (mock)
- Tap "Tout voir" (voyages) → écran liste complète (`CTRL — Voyages du jour` du Figma)

**Critères d'acceptation** :
- [ ] Compteurs cohérents avec les données serveur
- [ ] Liste voyages du jour = uniquement ceux assignés au contrôleur connecté
- [ ] Mode offline : liste + compteurs affichent le dernier snapshot avec indication de fraîcheur

---

### 5.3 Écran 3 — Détail voyage + Préparation du contrôle

**Sources Figma** : `detail-voyage-controleur`, `CTRL — Préparation du contrôle`

**Data affichée (Détail Voyage)** :
- Route (Bamako → Abidjan), statut (En cours)
- Départ prévu (07:30) + Arrivée prévue (13:45), n° voyage (VY-2024-001)
- **Bloc Bus** : immatriculation, modèle, capacité
- **Bloc Chauffeur** : nom, matricule, expérience, téléphone, rating
- **Bloc Escales** : timeline verticale des escales avec statut (Effectué / En cours / À venir) + heures prévues et effectuées
- **Bloc Passagers** : compteurs Billets (32), Embarqués (20), Absents (8), Cas particuliers (2), Sièges vides (12)

**Data affichée (Préparation du contrôle)** :
- Bloc info voyage (route, date, chauffeur, effectifs)
- **Checklist en 7 items** avec statut (Prêt / Attente) :
  1. Manifeste des passagers téléchargé
  2. Bordereau des colis téléchargé
  3. Données disponibles hors ligne
  4. Appareil mobile autorisé
  5. Scanner QR disponible
  6. Batterie suffisante
  7. Dernière synchronisation effectuée
- **Bloc "Mode Hors-ligne Activé"** : X passagers enregistrés localement, Y colis enregistrés localement, dernière synchro, stockage (124 Mo disponibles)
- **Bouton "Synchroniser les données"** (déclenche le download manifeste complet)

**Appels API** :
- `GET /trips/{id}` → détail voyage
- `GET /trips/{id}/manifest` → payload complet (passagers + colis + politique bagages + plan de sièges + escales)
- `POST /trips/{id}/control/open` → crée `ControlSession`, retourne `session_id`
- Vérifications locales (batterie, permission caméra, stockage dispo) → SDK RN

**Comportement offline** :
- L'écran de préparation **exige un réseau** pour fonctionner (télécharger manifeste)
- Une fois manifeste téléchargé + session ouverte, l'app peut passer offline en toute autonomie
- Snapshot manifeste stocké en SQLite avec timestamp `snapshot_taken_at`

**Interactions** :
- Bouton "Synchroniser les données" → download manifeste, met à jour la checklist
- Une fois checklist 100% verte + session ouverte → bouton "Démarrer le contrôle" → écran 4

**Critères d'acceptation** :
- [ ] Manifeste téléchargé et disponible offline (test : couper le réseau après download, l'écran 4 doit fonctionner)
- [ ] Session de contrôle créée côté serveur avec `opened_at`
- [ ] Checklist bloque le démarrage si items critiques (manifeste, batterie, caméra) manquants

---

### 5.4 Écran 4 — Contrôles (multi-onglets + sous-écrans)

**Sources Figma** (nombreuses) :
- Onglets : `Contrôles`, `Contrôles - Colis`, `Contrôles` (Scanner QR)
- Sous-écrans passagers : `Scan embarquement`, `Scan Success`, `Scan Warning`, `Accueil - v2` (billet expiré), `Accueil - v2` (QR invalide), `Saisie manuelle - Formulaire`, `Saisie manuelle - Billet trouvé`, `Saisie manuelle - Aucun résultat`, `Saisie manuelle - Plusieurs résultats`, `detail-billet-badges`, `fiche-passager`, `confirmation-embarquement`
- Sous-écrans colis : `Scan Colis`, `Saisie manuelle - Formulaire` (colis), `Colis Detail`, `controle-colis-validation`
- Sous-écrans cas spéciaux : `cas-special-controleur`
- Sous-écrans anomalies : `CTRL — Liste des anomalies`, `CTRL — Détail anomalie`
- Sous-écrans incidents : `incident-nouveau`, `incident-detail`

C'est **le cœur fonctionnel** de l'app. Découpage en 3 onglets :

#### 5.4.A Onglet Passagers

**Data affichée** :
- Barre de recherche (par nom / téléphone / siège)
- Chips filtres : Tous (32) / Embarqués (20) / En attente (12)
- **3 compteurs cliquables** : Embarqués (20) / En attente (12) / Total (32)
- Liste passagers : avatar (initiales), nom, siège, heure embarquement si embarqué, badge statut (Embarqué / En attente)

**Interactions par passager** :
- Tap → écran `fiche-passager` (détail complet avec DDN, CNI, téléphone, siège, heure embarquement, trajet)
- Bouton "Valider embarquement" → event `reservation_board` en queue offline
- Bouton "Cas spécial" → écran `cas-special-controleur`

**Appels API** :
- Lecture initiale = snapshot local (pas d'appel)
- Actions = events poussés dans la queue locale, uploadés via `POST /control-events/batch` dès reconnexion

#### 5.4.B Onglet Colis

**Data affichée** :
- Chips filtres : Tous (13) / Vérifiés (8) / Anomalies (1) / En attente (4)
- Liste colis : référence (COL-001), destinataire, poids, destination, badge statut

**Interactions par colis** :
- Tap → écran `Colis Detail` (numéro, destination, destinataire, téléphone, poids, photos de contrôle, notes)
- Bouton "Valider le colis" → écran `controle-colis-validation` avec animation succès → event `parcel_verify`
- Bouton "Ajouter une photo" → capture caméra → upload différé
- Bouton "Signaler anomalie" → écran anomalie détaillée

**Appels API** :
- Lecture = snapshot local
- Photo = upload différé via `POST /uploads/presign` puis upload direct S3 dès reconnexion

#### 5.4.C Onglet Scanner QR

**Data affichée** :
- Overlay caméra live avec cadre QR + indicateur "Position idéale / Maintenez le téléphone stable"
- Bandeau inférieur avec 3 compteurs (Embarqués / En attente / Total) + liste des 2-3 derniers scans

**Comportement scan** :
- Cas 1 — **Billet valide** → écran `Scan Success` (bloc vert "Billet valide" + infos passager + siège) → bouton "Valider l'embarquement" → event `reservation_board`
- Cas 2 — **Déjà embarqué** → écran `Scan Warning` (bloc orange "Déjà embarqué" + comparaison horaires 1er scan vs actuel) → 2 boutons : "Retour au scan" (annule) ou "Cas particulier" → création automatique `Anomaly` sévérité `moderate` en background
- Cas 3 — **Billet expiré / autre voyage** → écran `Accueil - v2` "Billet expiré" → bouton "Refuser l'embarquement" (event `reservation_refuse`) ou "Cas particulier" (override contrôleur)
- Cas 4 — **QR non reconnu** → écran `Accueil - v2` "QR Code Invalide" avec causes possibles → bouton "Réessayer le scan" ou "Saisie manuelle"
- **Bouton flottant "Saisie manuelle"** toujours accessible (fallback offline si caméra HS ou QR abîmé)

#### 5.4.D Sous-écran Saisie manuelle

**Sources Figma** : `Saisie manuelle - Formulaire`, `Saisie manuelle - Billet trouvé`, `Saisie manuelle - Aucun résultat`, `Saisie manuelle - Plusieurs résultats`

**Data affichée** :
- Champ "Référence billet" (ex `Ex: TBP-BKT-2024-08412`)
- Séparateur "OU"
- Champs recherche avancée : Nom du passager, Téléphone, Numéro de siège
- Bouton "Rechercher le billet"

**Comportement** :
- Recherche s'effectue **sur le snapshot local d'abord** (offline-friendly)
- Si aucun résultat local et réseau dispo → tenter API `GET /reservations?trip_id=X&search=Y`
- Résultats :
  - 1 résultat → écran `Saisie manuelle - Billet trouvé` (bloc vert avec infos)
  - 0 résultat → écran `Aucun billet trouvé` (conseil + boutons Réessayer / Signaler incident)
  - N résultats → liste `Saisie manuelle - Plusieurs résultats`

#### 5.4.E Sous-écran Cas Spécial

**Sources Figma** : `cas-special-controleur`

**Data affichée** :
- Bloc passager en tête (nom, siège, billet, trajet, statut actuel)
- Section "Type de cas" : 4 radios exclusifs :
  1. Enfant moins de 4 ans (voyage gratuit)
  2. Accompagnant (accompagne un passager valide)
  3. Cas médical (nécessite assistance médicale)
  4. Autre (situation non listée)
- Commentaire (200 caractères max)
- Pièces jointes : photo ou document (JPG/PNG/PDF)
- **Section conditionnelle "Motifs de refus"** (si Autre + refus) : chips (Billet invalide, Fraude présumée, Comportement inapproprié, Document manquant, Autre motif)
- 2 boutons : "Autoriser l'embarquement" (vert) / "Refuser l'embarquement" (rouge)

**Comportement** :
- Autoriser → event `reservation_special_case` avec `authorized: true`
- Refuser → event `reservation_refuse` avec `motif: xxx`
- Upload photos différé

#### 5.4.F Sous-écran Vente onboard (walk-in cash)

**⚠️ Pas d'écran Figma dédié** — à concevoir. Proposition d'écran minimal :

**Data affichée** :
- Bandeau "Nouvelle vente à bord"
- Champ Nom du passager (obligatoire)
- Champ Téléphone (obligatoire)
- Sélecteur siège : dropdown des sièges marqués "libre" dans le snapshot local
- Sélecteur destination : dropdown des `Waypoint` restants (escales natives FLB à partir de l'escale actuelle)
- Prix calculé automatiquement (tarif fixe route ou fixe voyage — cf §6)
- Bouton "Encaisser cash et valider"

**Comportement** :
- Event `onboard_sale` en queue avec `{passenger, seat, stop, price_xof, payment_method: cash, client_uuid}`
- Le serveur peut **rejeter** à la sync si le siège est pris entre-temps (cf §3.1)
- Ajout local dans le compteur "Cash collecté" (visible à la clôture)

#### 5.4.G Sous-écran Anomalies

**Sources Figma** : `CTRL — Liste des anomalies`, `CTRL — Détail anomalie`

**Data affichée (Liste)** :
- Chips filtres : Toutes (12) / Passagers / Colis / En attente / Résolue
- Liste d'anomalies : titre (Billet déjà scanné), passager/colis concerné, référence, statut/sévérité
- Compteurs top : Total (12) / En attente (5) / Critiques (2) / Résolues (5)

**Data affichée (Détail)** :
- Bloc sévérité (Critique) + statut (À traiter) + titre + référence + date/heure
- Bloc voyage + passager
- Description
- Bloc appareil de scan + heure + tentative n° + historique des scans (première validation, 2ᵉ scan détecté)
- Boutons : "Signaler un incident" (escalade dispatcher) / "Refuser l'embarquement"

**Comportement** :
- Les anomalies sont **majoritairement créées automatiquement** par le backend (double-scan, siège conflit) ou par actions contrôleur (refus, cas spécial)
- Le contrôleur les consulte et peut escalader en incident

#### 5.4.H Sous-écran Incidents

**Sources Figma** : `incident-nouveau`, `incident-detail`

**Data affichée (Nouveau)** :
- Titre (bref résumé)
- Type d'incident (dropdown : Bagages, Comportement, Sécurité, Panne, Autre)
- Sévérité : 3 chips (Faible / Modéré / Critique)
- Description
- Photos (jusqu'à N)
- Position GPS auto-détectée (issue du device, cf note §2.3.D-bis)
- Toggle "Notifier le dispatcher"
- Bouton "Envoyer le rapport"

**Data affichée (Détail)** :
- Statut (En cours / Résolu)
- Bloc infos (titre, bagages, sévérité, signalé par, date/heure)
- Description longue
- Photos
- Position GPS + adresse résolue
- Timeline : Incident signalé → Dispatcher notifié → En cours de traitement → Résolu
- Bouton "Marquer comme résolu"

**Comportement** :
- Upload photos différé
- Envoi en queue offline
- Notification dispatcher = mock V1 (log serveur), FCM V2

**Note sur la position GPS de l'incident** :
Le doc de reprise indique que "la position GPS ne peut être mise à jour QUE par l'app Navigator authentifiée avec un token chauffeur" côté modèle `Driver`. Pour l'incident, la position GPS est **stockée sur l'incident lui-même** (pas sur le `Controller`), ce qui ne conflicte pas avec cette règle. **À valider en code review**.

---

### 5.5 Écran 5 — Clôture du contrôle

**Sources Figma** : `Chauff - Accueil` (mal labellisé — c'est en fait le récap embarquement), `CTRL — Clôture du contrôle`, `manifeste-de-cloture`

**Data affichée (Récap embarquement)** :
- Titre "Manifeste clôturé — Bamako → Abidjan — 07:35"
- 2 compteurs macro : Total billets (32) / Embarqués (28)
- 2 compteurs secondaires : Absents (4) / Cas spéciaux (1)
- Progress bar "Statistiques taux d'embarquement 87.5%"
- Bouton "Retour au tableau de bord"

**Data affichée (Clôture détaillée)** :
- Route + n° voyage + bus + chauffeur + contrôleur + date/heure départ
- **Statistiques passagers** : 45 attendus / 42 embarqués / 2 absents / 1 cas spécial
- **Statistiques colis** : 32 attendus / 30 validés / 1 absent / 2 anomalies signalées
- Alerte "1 passager non contrôlé et 2 anomalies non résolues"
- **Bloc Synthèse sécurité & anomalies** : compteurs Incident déclaré (1), Anomalies résolues (5), Anomalies non résolues (1), Anomalie critique (1)
- **Bloc État de synchronisation** : Réseau Connecté / Dernière synchro il y a 2 min (5 en attente)
- 2 boutons : "Voir les anomalies" / "Retour aux contrôles"

**Data affichée (Manifeste)** :
- Récapitulatif sièges : Capacité totale (45), Billets vendus (32), Embarqués (20), Sièges vides (13), Absents (12), Cas particuliers (2)
- **Plan visuel des sièges** (schéma du bus, sièges colorés selon statut)
- Liste colis (référence, destinataire, poids, statut)

**Data ajoutée V1 (non dans Figma) — Réconciliation cash** :
- Bloc "Ventes à bord" : liste des ventes onboard avec passager + montant + siège
- Total collecté cash (ex : 45 000 XOF)
- Champ "Montant remis à l'agence" (le contrôleur saisit)
- Différence calculée automatiquement
- **Bloc "Actions en attente de synchronisation"** : compteur, bouton "Forcer la synchronisation"
- Bouton "Clôturer le voyage" (grisé tant que queue non vide)

**Appels API** :
- `POST /trips/{id}/control/close` avec `{summary: {passengers: {...}, parcels: {...}, cash: {...}, discrepancies: [...]}}`
- Serveur vérifie cohérence et renvoie `{closed_at, discrepancies_server_side}`

**Comportement offline** :
- **La clôture EXIGE d'être en ligne** — safeguard métier : on ne clôt pas un voyage avec des events non synchronisés
- Si queue non vide → bouton "Clôturer" grisé + message "Synchronisez d'abord vos N actions en attente"

**Critères d'acceptation** :
- [ ] Session `closed_at` renseignée côté serveur
- [ ] Écran de succès affiché
- [ ] Toutes les données du voyage disponibles en lecture après clôture (pas d'écrasement du local)
- [ ] Impossible de clôturer avec queue non vide

---

### 5.6 Overlay permanent — Centre de synchronisation

**Sources Figma** : `CTRL — Centre de synchronisation`

**Accessibilité** : accessible depuis n'importe quel écran via badge "N actions en attente" en header, ou depuis Paramètres.

**Data affichée** :
- Bandeau connexion (Connecté au réseau / Hors ligne) + dernière synchro
- Compteur "8 actions en attente" + "1 échec" + "45 Mo utilisés"
- **Progress bar globale de synchronisation** (87% — 35/40)
- **Statut par catégorie** :
  1. Contrôles passagers : 28/32
  2. Contrôles colis : 5/5 complet
  3. Anomalies : 2/3
  4. Incidents : Aucune donnée (0/0)
  5. Validations exceptionnelles : Aucune donnée
- **Dernières actions locales** : liste des N derniers events (Contrôle passager BIL-4521, Anomalie créée ANO-0234, Contrôle colis COL-1234 avec échec + bouton Réessayer)

**Interactions** :
- Bouton "Réessayer" sur action en échec → retry ciblé de cet event
- Bouton "Forcer la synchro" global (rare, débug)

**Comportement** :
- Sync automatique en background dès qu'un réseau est détecté (`NetInfo` RN)
- Retry exponentiel sur échec réseau (1s, 2s, 4s, 8s, 16s, 32s → puis abandon jusqu'à action user)
- Idempotence garantie par `client_uuid` unique par event

---

## 6. Modèle de tarification V1

Pour supporter la vente onboard, il faut un moyen de calculer un prix.

**Décision V1 simple** : tarif fixe **par route** (pas par distance dynamique).

- Modèle `Route` (Fleetbase natif) étendu avec champ custom `default_price_xof` (ou table pivot `RoutePrice { route_id, origin_stop_id, destination_stop_id, price_xof }` pour tarification par segment)
- Ex : Bamako→Abidjan = 25 000 XOF ; Bamako→Yamoussoukro = 18 000 XOF

**Politique bagages** (rappel décision D) :
- Par voyage OU par route : `LuggagePolicy { free_luggage_count: 1, extra_luggage_price_xof: 2000, weight_limit_kg: 20, over_weight_price_per_kg: 500 }`
- Appliquée à la vente onboard de colis liés à un passager

**Backlog V2** : moteur tarifs zones, promotions, cartes fidélité.

---

## 7. Contraintes techniques transverses

### 7.1 Sécurité
- Tokens stockés en Keychain / EncryptedSharedPreferences uniquement
- CNI passager chiffrée au repos (colonne chiffrée MySQL ou libsodium application-side)
- Audit log sur tout accès à `passenger.id_card_*` (qui, quand, pour quelle raison)
- HTTPS obligatoire en prod (Caddyfile déjà configuré côté Fleetbase)

### 7.2 Idempotence
- **Tous les endpoints d'écriture** requièrent header `Idempotency-Key: <uuid v4>`
- Table backend `idempotency_keys` : `{key UNIQUE, user_id, endpoint, response_body, created_at}`
- TTL 7 jours

### 7.3 Sync offline — protocole
- Événements offline en queue locale SQLite : `client_uuid, session_id, event_type, target_type, target_id, payload_json, created_at_local, status (pending/syncing/sent/failed), retry_count, last_error`
- Upload batch via `POST /control-events/batch` (max 100 events par requête)
- Serveur traite **transactionnellement par event**, retourne verdict individuel
- Retry exponentiel sur échec réseau, arrêt sur échec métier (attention utilisateur)

### 7.4 Stockage local mobile (recommandations à valider par équipier RN)
- **SQLite via `react-native-quick-sqlite`** (performance) ou **WatermelonDB** (ORM offline-first)
- Chiffrement DB local via SQLCipher
- Quota max : 200 Mo par voyage (photos incluses), purge automatique après clôture + délai grâce 7 jours

### 7.5 Multi-tenant
- Toutes les tables `toupac-voyage` ont `company_id` avec index
- Tous les endpoints scopent implicitement via `Auth::user()->company_uuid` (middleware Fleetbase existant)

---

## 8. Contraintes réglementaires

### 8.1 RGPD-like (Sénégal : loi 2008-12 sur la protection des données)
- Consentement passager lors de la création de son compte (côté app CLIENT — hors scope V1)
- Droit d'accès et de rectification
- Durée de rétention : à définir avec le client, hypothèse par défaut 3 ans après dernier voyage
- CNI stockée chiffrée + audit log

### 8.2 Compagnies de transport
- Manifeste de clôture doit être **exportable** (PDF ou impression) pour remise aux autorités si contrôle route
- **V1** : bouton "Exporter manifeste" produit un PDF côté serveur, téléchargeable après clôture

---

## 9. Métriques de succès V1 (démo 19 sept)

Le MVP est considéré comme réussi si les scénarios suivants passent en démo :

**Scénario 1 — Voyage nominal**
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

**Scénario 2 — Gestion anomalie**
1. Contrôleur scanne un billet valide (embarquement OK)
2. **Rescanne le même billet** → écran "Déjà embarqué"
3. Anomalie créée automatiquement (visible dans Liste anomalies)
4. Contrôleur consulte le détail anomalie, marque "signaler incident" → incident créé

**Scénario 3 — Conflit siège onboard vs app**
1. Contrôleur télécharge manifeste, part en mode offline
2. Un test manuel côté console : marquer siège A14 comme vendu à un autre passager
3. Contrôleur vend siège A14 onboard offline
4. Repasse en ligne → sync
5. Notification "Conflit détecté sur siège A14"
6. Contrôleur choisit "Réattribuer un siège libre" → nouvelle vente sur siège A15

---

## 10. Découpage macro du sprint (rappel)

| Sprint | Dates | Livrable |
|---|---|---|
| S1 — Cadrage & Contrat API | 20-26 août | MVP-SPEC + ERD + OpenAPI + Scaffold + Modèles/migrations |
| S2 — Backend endpoints | 27 août - 05 sept | 19 endpoints implémentés + tests feature + seeders démo |
| S3 — Intégration & hardening backend / dev RN en parallèle | 06-14 sept | Corrections retours équipier RN + optimisations perf |
| S4 — Tests end-to-end offline + dry-run démo | 15-18 sept | Scénarios 1-2-3 verts + rehearsal démo |
| **Jour J** | **19 sept** | Démo client |

---

## 11. Répartition des rôles

| Rôle | Personne | Responsabilité |
|---|---|---|
| Lead / Architecte backend | Freelance (moi) | Backend `toupac-voyage`, API, migrations, tests, supervision RN |
| Dev mobile RN | Équipier de l'équipe | App CONTRÔLEUR React Native (TypeScript strict, offline-first) |
| Product owner / Client | Société sénégalaise | Validation des specs, arbitrages fonctionnels |

Points de synchro proposés :
- **Standup async quotidien** (Slack ou équivalent, 3 lignes matin)
- **Design review lundi** (30 min) sur les évolutions API et blockers
- **Demo intermédiaire vendredi** (fin de chaque sprint)

---

## 12. Backlog V2 récapitulatif

Priorité à retravailler avec le client post-jalon 19 sept :

- App CLIENT (billetterie mobile + envoi colis + upload identité)
- App CHAUFFEUR (fork Navigator App adapté)
- Console AGENCE dédiée (dashboard vendeur comptoir)
- Vente onboard **mobile money** (Wave, OM, Free Money, MTN MoMo, agrégateur)
- Push notifications réelles (décision provider + intégration)
- Auth biométrique locale
- Édition de politique bagages en console + moteur de règles avancé
- Reporting BI (dashboards custom, exports)
- Multi-langue (EN, WO)
- OSRM auto-hébergé + intégration Chauffeur
- WhatsApp Business API
- Extension **`toupac-payments-aggregator`** (dépendance vente onboard MoMo)
- Extension **`toupac-notifications-whatsapp`**

---

## 13. Annexes

### 13.1 Références Figma
Les écrans Figma cités sont issus des 2 exports partagés le 19 août 2026. Note : URL Figma source à obtenir du client pour référence permanente.

### 13.2 Références Fleetbase
- Extension development : https://fleetbase.io/docs/extension-development
- API Reference : https://fleetbase.io/docs/api
- Universe Service (hooks, menu, extension manager) : https://fleetbase.io/docs/extension-development/universe/overview
- Recette "Adding a Payment Gateway Driver" (utile V2 pour MoMo) : https://fleetbase.io/docs/extension-development/recipes

### 13.3 Fichiers produits pendant le sprint 1
- `TOUPAC-VOYAGE-MVP-SPEC.md` (ce document — J1, patché v1.1)
- `TOUPAC-VOYAGE-ERD.md` (J2, réécrit v1.1)
- `TOUPAC-VOYAGE-ARCHITECTURE.md` (J2, réécrit v1.1)
- `TOUPAC-VOYAGE-CHANGELOG.md` (log des changements v1.0 → v1.1)
- `TOUPAC-VOYAGE-DEV-BRIEF.md` (brief équipier RN)
- `TOUPAC-VOYAGE-OPENAPI.yaml` + collection Postman (J3)
- `packages/toupac-voyage/` (scaffold — J4)
- Modèles + migrations + factories + seeders (J5)

Après scaffold, les documents `TOUPAC-VOYAGE-*.md` seront déplacés dans `packages/toupac-voyage/docs/` via `git mv`.

---

**Fin du document — Version 1.1**

**Prochaine action** : validation de ce document par le lead, puis passage à J3 (OpenAPI + collection Postman).
