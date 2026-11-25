# Documentation détaillée des API REST — MedGrid (`/api/v1`)

## 1. Principes généraux & Routage
- **Point d'entrée unique** : la console web (et tout client externe) appelle uniquement `medgrid-gateway` (port exposé `8080`), qui valide le jeton JWT (RS256), applique la limitation de débit (Redis), gère CORS et route `/api/v1/**` vers les services internes.
- **Format d'échange** : JSON (`application/json`) sur HTTPS/HTTP interne, sauf export PDF (`application/pdf`) et flux temps réel (`text/event-stream` en SSE).
- **Rôles RBAC** : `ACCUEIL`, `MEDECIN`, `INFIRMIER`, `FACTURATION`, `ADMIN` (contrôlés à la gateway et dans chaque service).

---

## 2. En-têtes HTTP standards
- `Authorization: Bearer <access_token>` : JWT signé RS256 (durée de vie 15 min ; refresh token de 8 h).
- `X-Correlation-Id: <uuid>` : identifiant de corrélation généré par la gateway s'il est absent, injecté dans les logs JSON et propagé aux services et événements.
- `Idempotency-Key: <uuid>` : obligatoire sur les opérations de création critiques (`POST /api/v1/admissions`, `POST /api/v1/factures`) afin d'éviter les doublons en cas de nouvelle tentative.
- `Last-Event-ID: <event_id>` : utilisé sur le flux SSE des alertes (`GET /api/v1/alertes/stream`) pour reprendre après reconnexion.

---

## 3. Format d'erreur unifié — RFC 7807 (`application/problem+json`)
Chaque service REST expose un `@RestControllerAdvice` retournant des erreurs conformes à **RFC 7807** avec un `traceId` pour faciliter le diagnostic distribué :

```json
{
  "type": "https://medgrid.example/errors/conflict",
  "title": "Conflit métier",
  "status": 409,
  "detail": "Aucun lit disponible dans l'unité demandée.",
  "instance": "/api/v1/admissions",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

---

## 4. Catalogue détaillé des endpoints par service

### A. `auth-service` (Authentification & Comptes — Responsable : Membre 1)
- **`POST /api/v1/auth/login`**
  - **Rôles** : Public.
  - **Corps attendu** : `{ "username": "string", "password": "string" }`.
  - **Réponses** :
    - `200 OK` : retourne l'`accessToken` (JWT 15 min) et le `refreshToken` (8 h).
    - `401 Unauthorized` : identifiants invalides.
    - `423 Locked` : compte verrouillé après 5 échecs consécutifs.
- **`POST /api/v1/auth/refresh`** *(et gestion de session / logout)*
  - **Rôles** : Utilisateur authentifié.
  - **Réponses** : `200 OK` (nouveaux jetons) ou `401 Unauthorized`.

### B. `admission-service` (Patients & Parcours d'admission — Responsable : Membre 2)
- **`POST /api/v1/patients`**
  - **Rôles** : `ACCUEIL`.
  - **Réponses** : `201 Created`, `400 Bad Request` (validation), `409 Conflict` (doublon détecté).
- **`GET /api/v1/patients?nom=&page=&size=`**
  - **Rôles** : `ACCUEIL`, `MEDECIN`, `INFIRMIER`.
  - **Réponses** : `200 OK` — liste paginée, filtrable et triable.
- **`POST /api/v1/admissions`**
  - **Rôles** : `ACCUEIL`.
  - **En-tête requis** : `Idempotency-Key`.
  - **Comportement** : types d'admission (`urgence`, `programmée`, `consultation`) ; statuts possibles (`EN_ATTENTE`, `VALIDEE`, `EN_COURS`, `CLOTUREE`, `ANNULEE`, `EN_ATTENTE_MUTUELLE`) ; orchestre la réservation de lit (gRPC) puis la vérification des droits (SOAP) avec compensation saga si refus.
  - **Réponses** :
    - `201 Created` : admission validée et lit réservé.
    - `202 Accepted` : statut `EN_ATTENTE_MUTUELLE` (mode dégradé si la mutuelle est indisponible, avec rattrapage automatique).
    - `409 Conflict` : aucun lit disponible ou conflit d'état.
    - `503 Service Unavailable` : indisponibilité sans repli possible.
- **`GET /api/v1/admissions/{id}`**
  - **Rôles** : `ACCUEIL`, `MEDECIN`.
  - **Réponses** : `200 OK`, `404 Not Found`.
- **`POST /api/v1/admissions/{id}/cloture`**
  - **Rôles** : `MEDECIN`.
  - **Comportement** : libère le lit et publie l'événement `admission.cloturee`.
  - **Réponses** : `200 OK`, `409 Conflict` (statut incompatible).
- **`GET /api/v1/admissions/{id}/constantes`**
  - **Rôles** : `MEDECIN`, `INFIRMIER`.
  - **Réponses** : `200 OK` — dernières mesures / relais de la télémétrie.

### C. `planning-service` (Rendez-vous & Créneaux — Responsable : Membre 2)
- **`POST /api/v1/rendez-vous`**
  - **Rôles** : `ACCUEIL`.
  - **Réponses** : `201 Created`, `409 Conflict` (créneau déjà pris — contrainte en base et verrou optimiste).
- **`GET /api/v1/medecins/{id}/creneaux?date=`**
  - **Rôles** : `ACCUEIL`.
  - **Réponses** : `200 OK` — créneaux et disponibilités du médecin pour la date ciblée.

### D. `alert-service` (Alertes cliniques & Flux SSE — Responsable : Membre 3)
- **`GET /api/v1/alertes/stream`**
  - **Rôles** : `MEDECIN`, `INFIRMIER`.
  - **Réponses** : `200 OK` (`text/event-stream`) — diffusion des alertes en temps réel avec reprise via `Last-Event-ID`.
- **`POST /api/v1/alertes/{id}/acquittement`**
  - **Rôles** : `INFIRMIER`.
  - **Comportement** : fait passer l'alerte du cycle `OUVERTE` à `ACQUITTEE` (puis `RESOLUE`) et mesure le temps de réaction.
  - **Réponses** : `200 OK`, `404 Not Found`.

### E. `billing-service` (Facturation & Paiements — Responsable : Membre 4)
- **`POST /api/v1/factures`**
  - **Rôles** : `FACTURATION`.
  - **Comportement** : génère la facture (lignes d'actes, part mutuelle, reste à charge en `BigDecimal`, cycle `BROUILLON`, `EMISE`, `PAYEE_PARTIELLEMENT`, `PAYEE`, `REJETEE`).
  - **Réponses** : `201 Created`, `422 Unprocessable Entity` (admission non clôturée).
- **`GET /api/v1/factures/{id}/pdf`**
  - **Rôles** : `FACTURATION`.
  - **Réponses** : `200 OK` (`application/pdf`) — export PDF de la facture.

### F. `audit-service` (Journal d'audit — Responsable : Membre 1)
- **`GET /api/v1/audit?patient=&from=&to=`**
  - **Rôles** : `ADMIN`.
  - **Réponses** : `200 OK` — consultation paginée des accès aux dossiers (qui, quoi, quand, sur quel patient).

---

## 5. Règles anti-conflits lors du versionnement REST (`/api/v1`)
1. **Ne pas casser `/api/v1`** : toute modification sur un endpoint existant doit rester rétrocompatible (pas de suppression de champ, pas de renommage destructif, pas de nouveau champ obligatoire en requête).
2. **Nouvelle version majeure (`/api/v2`) uniquement en cas de rupture** : si un changement incompatible est indispensable, créer une route `/api/v2/...` et une spécification OpenAPI dédiée plutôt que de modifier `/api/v1` à l'aveugle.
3. **Revue croisée obligatoire** : toute modification de contrat OpenAPI passe par une Pull Request dédiée dans `/contracts/openapi/` relue par les consommateurs concernés avant d'écrire le code.
4. **Mocks dès le Sprint 2** : chaque producteur fournit un mock/stub de son service afin qu'aucun membre ne soit bloqué en attendant l'implémentation complète d'un autre service.