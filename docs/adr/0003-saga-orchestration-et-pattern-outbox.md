# ADR-003 : Saga d'admission et publication fiable par pattern Outbox

- **Statut** : Accepté
- **Date** : 2026-11-25
- **Auteur** : Membre 2
- **Relecteur** : Membre 1

## Contexte
La création d'une admission (`POST /api/v1/admissions`) implique de réserver un lit dans `telemetry-service` (gRPC), de vérifier les droits dans `insurance-service` (SOAP) et de publier l'événement `admission.creee` sur RabbitMQ. Aucune admission ne doit être perdue en cas de panne de la mutuelle ou du broker (EF-08, EF-09, ENF-07, ENF-08).

## Décision
- Implémenter une **Saga orchestrée** dans `admission-service` avec une table d'état persistante :
  1. Réservation atomique du lit via gRPC (deadline 800 ms, repli sur capacité nominale si télémétrie indisponible).
  2. Vérification des droits via SOAP (timeouts 1 s / 3 s, Circuit Breaker Resilience4j). Si refus métier, action de compensation immédiate pour libérer le lit. Si panne technique, passage au statut `EN_ATTENTE_MUTUELLE` (HTTP 202) et rattrapage planifié.
- Utiliser le **Transactional Outbox Pattern** : enregistrer les événements `admission.*` dans une table `outbox_events` au sein de la même transaction PostgreSQL que l'admission, puis un relai publie vers RabbitMQ.

## Alternatives envisagées
- **Transaction distribuée 2PC (XA)** : trop lourde, bloquante et incompatible avec des appels gRPC/SOAP hétérogènes.
- **Publication RabbitMQ directe après commit** : risque de perte d'événement si le broker est coupé (scénario 4).

## Conséquences
- **Positives** : cohérence éventuelle garantie, zéro perte de données en mode dégradé.
- **Négatives / Risques** : nécessité de dédupliquer côté consommateurs grâce à l'`eventId`.
