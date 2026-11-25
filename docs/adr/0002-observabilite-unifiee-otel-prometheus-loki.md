# ADR-002 : Observabilité distribuée (OpenTelemetry, Prometheus, Grafana, Loki)

- **Statut** : Accepté
- **Date** : 2026-11-25
- **Auteur** : Membre 1
- **Relecteur** : Membre 3

## Contexte
Dans une architecture distribuée mêlant REST, gRPC, SOAP et RabbitMQ, il est indispensable de suivre une requête de bout en bout (front-end jusqu'à `insurance-service`), de surveiller les temps de réponse (p95 < 300 ms) et de diagnostiquer les pannes en direct (ENF-09, ENF-10).

## Décision
- Instrumenter tous les services Spring Boot avec **Micrometer Tracing + OpenTelemetry** pour propager un `traceId` unique à travers HTTP, gRPC metadata, en-têtes SOAP et messages AMQP.
- Formater tous les logs applicatifs en **JSON** sur `stdout` (avec `traceId`, sans données médicales sensibles) et les collecter via **Promtail + Loki**.
- Exposer `/actuator/prometheus` sur chaque service et provisionner **9 tableaux de bord Grafana** (un par service + une vue globale) au démarrage.

## Alternatives envisagées
- **Logs texte locaux dans chaque conteneur** : inexploitables lors d'une panne multi-services.

## Conséquences
- **Positives** : corrélation immédiate entre métriques, traces et journaux lors des tests de chaos.
- **Négatives / Risques** : consommation mémoire supplémentaire sur le poste de développement, atténuée par le profil Docker Compose `observability`.
