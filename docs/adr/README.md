# Architecture Decision Records (ADR) — MedGrid

Ce dossier documente les décisions d'architecture majeures du projet MedGrid (exigence ENF-15).
Chaque ADR doit être relu par un autre membre de l'équipe via Pull Request.

| ID | Titre | Statut | Auteur(s) |
| :--- | :--- | :--- | :--- |
| [ADR-001](0001-gateway-et-authentification-rs256.md) | API Gateway unique et authentification JWT RS256 | Accepté | Membre 1 |
| [ADR-002](0002-observabilite-unifiee-otel-prometheus-loki.md) | Observabilité distribuée (OpenTelemetry, Prometheus, Grafana, Loki) | Accepté | Membre 1 |
| [ADR-003](0003-saga-orchestration-et-pattern-outbox.md) | Saga d'admission et publication fiable par pattern Outbox | Accepté | Membre 2 |
| [ADR-004](0004-grpc-streaming-et-series-temporelles-mongodb.md) | Télémétrie haute fréquence via gRPC et séries temporelles MongoDB | Accepté | Membre 3 |
| [ADR-005](0005-soap-contract-first-mutuelle.md) | Intégration mutuelle en SOAP contract-first (XSD/WSDL) | Accepté | Membre 4 |
| [ADR-006](0006-facturation-asynchrone-et-idempotence.md) | Facturation événementielle asynchrone et idempotence | Accepté | Membre 4 |
