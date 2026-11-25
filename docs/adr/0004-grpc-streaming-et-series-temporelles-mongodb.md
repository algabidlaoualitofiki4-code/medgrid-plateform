# ADR-004 : Télémétrie haute fréquence via gRPC et séries temporelles MongoDB

- **Statut** : Accepté
- **Date** : 2026-11-25
- **Auteur** : Membre 3
- **Relecteur** : Membre 1

## Contexte
`telemetry-service` doit ingérer jusqu'à 200 mesures par seconde depuis le simulateur sur au moins 50 lits sans perte, diffuser les constantes en temps réel et garantir une réservation de lit sans concurrence (EF-17 à EF-22, ENF-06).

## Décision
- Utiliser **gRPC / Protobuf (`telemetry.v1`)** avec :
  - 3 RPC unaires (`CheckBedAvailability`, `ReserveBed`, `ReleaseBed`) protégés par un verrouillage atomique.
  - 1 RPC en streaming client (`IngestVitals`) permettant au simulateur d'envoyer un flux continu avec gestion de la contre-pression et écriture par lots.
  - 1 RPC en streaming serveur (`StreamVitals`) pour la diffusion temps réel.
- Stocker les constantes dans une **collection Time-Series MongoDB** avec index TTL (rétention 48 h pour les mesures brutes) et publier `mesure.recue` sur RabbitMQ.

## Alternatives envisagées
- **REST/JSON pour l'ingestion des constantes** : surcharge réseau et sérialisation trop coûteuse à 200 req/s.
- **PostgreSQL pour les séries temporelles** : moins adapté aux écritures rafales haute fréquence et à l'expiration TTL automatique.

## Conséquences
- **Positives** : faible latence (< 800 ms), contrat strict vérifié par `buf breaking`.
- **Négatives / Risques** : nécessite la génération des stubs Protobuf à la compilation Maven.
