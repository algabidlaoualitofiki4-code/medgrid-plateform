# ADR-005 : Intégration mutuelle en SOAP contract-first (XSD/WSDL)

- **Statut** : Accepté
- **Date** : 2026-11-25
- **Auteur** : Membre 4
- **Relecteur** : Membre 2

## Contexte
Le cahier des charges impose un service SOAP *contract-first* pour simuler l'interconnexion avec les organismes de mutuelle hérités (`VerifyCoverage`, `RequestPreApproval`, `GetClaimStatus`, `CloseClaim`) avec gestion fine des erreurs métiers et validation stricte des messages (EF-29 à EF-33).

## Décision
- Rédiger d'abord le schéma XML **`contracts/soap/insurance-v1.xsd`** (namespace `http://medgrid.example/insurance/v1`) définissant les requêtes, réponses et codes de faute typés (`INVALID_POLICY`, `CEILING_EXCEEDED`, `COVERAGE_REFUSED`, `INVALID_CLAIM`, `CLAIM_ALREADY_CLOSED`).
- Générer les classes Java via **JAXB / XJC** lors du build Maven et exposer le WSDL automatiquement avec **Spring Web Services** en activant la validation XSD entrante et sortante.

## Alternatives envisagées
- **Code-first (génération du XSD depuis les classes Java)** : interdit par le cahier des charges et expose les clients à des ruptures involontaires lors du refactoring Java.

## Conséquences
- **Positives** : contrat figé et validé formellement dès le Sprint 1 ; erreurs métiers structurées dans le `SOAPFault`.
- **Négatives / Risques** : verbosité XML plus élevée que JSON, maîtrisée par la génération automatique XJC.
