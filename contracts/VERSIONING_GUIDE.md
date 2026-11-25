# Guide de versionnement des contrats d'API — MedGrid

## 1. Principe général
Toute interface entre services est définie dans `/contracts` **avant** l'écriture du code métier et validée via Pull Request.
- Aucun code applicatif (Java, TypeScript, Docker) ne doit être mélangé dans une PR modifiant `/contracts`.
- Toute modification de contrat doit être revue par le ou les membres qui consomment ce contrat.

## 2. Règles de rétrocompatibilité (version v1)
- **REST (`/api/v1/**`)** :
  - Ne pas supprimer ni renommer un champ JSON existant.
  - Ne pas rendre obligatoire un nouveau paramètre ou champ.
  - Toutes les erreurs applicatives suivent le format RFC 7807 (`application/problem+json`) avec un `traceId`.
- **gRPC (`telemetry.v1`)** :
  - Ne jamais réutiliser ni modifier le numéro de tag d'un champ Protobuf existant.
  - Utiliser `reserved` ou `[deprecated = true]` si un champ est retiré/déprécié.
  - Vérification automatique en CI avec `buf breaking`.
- **SOAP (`insurance/v1`)** :
  - Approche *contract-first* basée sur `insurance-v1.xsd` (WSDL généré).
  - Tout nouvel élément optionnel doit avoir `minOccurs="0"`.
  - Les fautes utilisent exclusivement les codes définis dans le XSD (`INVALID_POLICY`, `CEILING_EXCEEDED`, `COVERAGE_REFUSED`, `INVALID_CLAIM`, `CLAIM_ALREADY_CLOSED`).
- **Événements RabbitMQ (`medgrid.events`)** :
  - Chaque message porte un `eventId` unique (idempotence des consommateurs) et un `traceId`.
  - Les consommateurs doivent ignorer les propriétés JSON inconnues.

## 3. En-têtes et métadonnées transverses
- `Authorization: Bearer <JWT>`
- `X-Correlation-Id: <UUID>` (propagé sur REST, gRPC metadata, SOAP et événements)
- `Idempotency-Key: <UUID>` (sur les créations critiques comme `POST /api/v1/admissions` et `POST /api/v1/factures`)
