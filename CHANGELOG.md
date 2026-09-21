# Changelog

## [1.0.0] — 2026-09-21
### Ajouté
- Parsers `auth.log` (syslog classique et ISO 8601) et `access.log` (Apache/Nginx *common* / *combined*), support `.gz`.
- 10 règles de détection mappées sur MITRE ATT&CK (SSH et web).
- Escalade automatique de la sévérité quand une attaque web reçoit une réponse 2xx.
- Rapports console (couleurs), JSON et HTML autonome.
- Configuration JSON des seuils et whitelist (IP ou CIDR).
- Générateur de journaux de démonstration et 31 tests unitaires.
