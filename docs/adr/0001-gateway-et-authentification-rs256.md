# ADR-001 : API Gateway unique et authentification JWT RS256

- **Statut** : Accepté
- **Date** : 2026-11-25
- **Auteur** : Membre 1
- **Relecteur** : Membre 2

## Contexte
MedGrid expose 9 microservices et un front-end React. Il faut un point d'entrée unique sécurisé, une gestion des rôles (ACCUEIL, MEDECIN, INFIRMIER, FACTURATION, ADMIN), une limitation de débit et la capacité de continuer à valider les sessions même si `auth-service` redémarre (ENF-01, ENF-02, scénario de panne 8).

## Décision
- Utiliser **Spring Cloud Gateway** (`medgrid-gateway`, port 8080) comme unique point d'entrée exposé hors du réseau Docker interne `backend`.
- Signer les JWT dans `auth-service` avec une paire de clés asymétriques **RS256** (durée de vie 15 min, refresh token 8 h) et exposer la clé publique via un endpoint **JWKS**.
- Mettre en cache la clé publique dans `medgrid-gateway` et propager l'identité ainsi que l'en-tête `X-Correlation-Id` vers tous les services aval.
- Appliquer le rate limiting par utilisateur via **Redis**.

## Alternatives envisagées
- **JWT symétrique (HS256)** : oblige à partager le secret de signature entre tous les microservices, augmentant la surface d'exposition.
- **Validation synchrone à chaque requête auprès d'auth-service** : crée un point de défaillance unique (SPOF) et augmente la latence.

## Conséquences
- **Positives** : si `auth-service` tombe, les jetons déjà émis restent valides (scénario 8) ; isolation réseau stricte.
- **Négatives / Risques** : nécessite une gestion propre de la révocation via les refresh tokens courts.
