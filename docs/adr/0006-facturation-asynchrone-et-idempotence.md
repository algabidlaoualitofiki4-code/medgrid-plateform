# ADR-006 : Facturation événementielle asynchrone et idempotence

- **Statut** : Accepté
- **Date** : 2026-11-25
- **Auteur** : Membre 4
- **Relecteur** : Membre 2

## Contexte
À la clôture d'une admission par le médecin, une facture doit être générée automatiquement en calculant la part mutuelle et le reste à charge, tout en supportant l'arrêt temporaire de `billing-service`, la livraison en double d'un événement RabbitMQ ou l'indisponibilité de la mutuelle (EF-34 à EF-37, scénarios de panne 6 et 9).

## Décision
- Déclencher la création de facture par abonnement asynchrone à l'événement RabbitMQ **`admission.cloturee`** (file durable avec Dead-Letter Queue).
- Garantir l'**idempotence** en enregistrant chaque `eventId` reçu dans une table `processed_events` dans la même transaction que la création de la facture, empêchant toute double facturation.
- Effectuer tous les calculs monétaires en **`BigDecimal`** avec règles d'arrondi explicites (`HALF_UP`).
- Si `insurance-service` est indisponible lors de l'appel SOAP `CloseClaim`, enregistrer la facture au statut `BROUILLON` et planifier une reprise automatique via **Quartz**.

## Alternatives envisagées
- **Appel REST synchrone depuis `admission-service` vers `billing-service` à la clôture** : couple fortement la clôture médicale à la disponibilité de la facturation.

## Conséquences
- **Positives** : résilience totale aux coupures de `billing-service` (scénario 6) et garantie d'unicité des factures (scénario 9).
- **Négatives / Risques** : création de facture légèrement différée (quelques millisecondes), acceptable métier.
