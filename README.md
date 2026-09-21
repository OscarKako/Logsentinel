# 🛡️ LogSentinel

[![CI](https://github.com/VOTRE-UTILISATEUR/logsentinel/actions/workflows/ci.yml/badge.svg)](https://github.com/VOTRE-UTILISATEUR/logsentinel/actions/workflows/ci.yml)
![Python](https://img.shields.io/badge/python-3.9%2B-blue)
![Dépendances](https://img.shields.io/badge/d%C3%A9pendances-0-brightgreen)
![Licence](https://img.shields.io/badge/licence-MIT-green)

**LogSentinel** est un système de détection d'intrusions léger (HIDS) écrit en Python pur. Il analyse les journaux **SSH** (`auth.log`) et **web** (`access.log` Apache/Nginx) pour identifier les attaques courantes, les classe par sévérité, les associe aux techniques **MITRE ATT&CK** et produit des rapports console, JSON et HTML.

> Projet *blue team* : conçu pour la défense, l'apprentissage de l'analyse de logs et la réponse à incident.

---

## ✨ Fonctionnalités

- **Zéro dépendance** : uniquement la bibliothèque standard Python.
- **Parsers robustes** : syslog classique et ISO 8601, formats web *common* et *combined*, guillemets échappés, IPv6, fichiers `.gz` issus de logrotate.
- **10 règles de détection** avec fenêtres glissantes en O(n).
- **Détection de compromission** : une connexion réussie après une force brute déclenche une alerte `CRITICAL`.
- **Priorisation intelligente** : une attaque web ayant reçu une réponse `2xx` voit sa sévérité automatiquement relevée.
- **Anti-évasion** : double décodage URL des charges utiles (`%252e%252e%252f` → `../`).
- **Rapports** : console colorée, JSON (intégrable à un SIEM) et HTML autonome.
- **Automatisable** : option `--fail-on` qui renvoie un code de sortie `2` pour les tâches cron ou la CI.
- **Configurable** : seuils et whitelist (IP ou CIDR) via un fichier JSON.

## 🎯 Règles de détection

| Règle | Sévérité | MITRE ATT&CK | Description |
|---|---|---|---|
| `SSH_SUCCESS_AFTER_BRUTE_FORCE` | CRITICAL | [T1078](https://attack.mitre.org/techniques/T1078/) | Connexion réussie après de nombreux échecs depuis la même IP |
| `SSH_BRUTE_FORCE` | HIGH | [T1110.001](https://attack.mitre.org/techniques/T1110/001/) | N échecs d'authentification dans une fenêtre de temps |
| `SSH_PASSWORD_SPRAYING` | MEDIUM | [T1110.003](https://attack.mitre.org/techniques/T1110/003/) | Nombreux comptes différents testés depuis une IP |
| `WEB_COMMAND_INJECTION` | CRITICAL | [T1190](https://attack.mitre.org/techniques/T1190/) | `;id`, `\|whoami`, `$(uname)`... |
| `WEB_LOG4SHELL` | CRITICAL | [T1190](https://attack.mitre.org/techniques/T1190/) | `${jndi:...}` dans l'URL, le User-Agent ou le Referer |
| `WEB_SQL_INJECTION` | HIGH | [T1190](https://attack.mitre.org/techniques/T1190/) | `UNION SELECT`, `' OR '1'='1`, `SLEEP()`... |
| `WEB_PATH_TRAVERSAL` | HIGH | [T1190](https://attack.mitre.org/techniques/T1190/) | `../`, `/etc/passwd`, `win.ini`... |
| `WEB_REMOTE_FILE_INCLUSION` | HIGH | [T1190](https://attack.mitre.org/techniques/T1190/) | `php://filter`, inclusion d'URL distante |
| `WEB_XSS` | MEDIUM | [T1190](https://attack.mitre.org/techniques/T1190/) | `<script>`, `onerror=`, `javascript:`... |
| `WEB_SENSITIVE_FILE` | MEDIUM | [T1083](https://attack.mitre.org/techniques/T1083/) | `.env`, `.git/`, `wp-config.php`, `*.sql`, `*.bak`... |
| `WEB_SCANNER_DETECTED` | MEDIUM | [T1595.002](https://attack.mitre.org/techniques/T1595/002/) | sqlmap, nikto, nmap, gobuster, nuclei, wpscan... |
| `WEB_DIRECTORY_ENUMERATION` | MEDIUM | [T1595.003](https://attack.mitre.org/techniques/T1595/003/) | Rafale d'erreurs 404 |
| `WEB_REQUEST_FLOOD` | MEDIUM | [T1499](https://attack.mitre.org/techniques/T1499/) | Volume de requêtes anormal depuis une IP |

## 🚀 Installation

```bash
git clone https://github.com/VOTRE-UTILISATEUR/logsentinel.git
cd logsentinel
python -m venv .venv && source .venv/bin/activate   # Windows : .venv\Scripts\activate
pip install -e .
```

## 🧪 Démarrage rapide (démo)

Le projet inclut un générateur de journaux réalistes mêlant trafic légitime et 9 scénarios d'attaque :

```bash
logsentinel generate-samples --out samples
logsentinel analyze --auth samples/auth.log --web samples/access.log --year 2026 --html rapport.html
```

Extrait de la sortie :

```
=== LogSentinel 1.0.0 — rapport d'analyse ===

Événements SSH : 81 (échecs : 52, succès : 21, utilisateurs invalides : 8)
Requêtes web   : 1130 depuis 68 IP distinctes {'2xx': 958, '3xx': 79, '4xx': 82, '5xx': 11}
Alertes        : CRITICAL=6  HIGH=3  MEDIUM=6  LOW=0

 CRITICAL  #1   SSH_SUCCESS_AFTER_BRUTE_FORCE  [T1078]  source : 203.0.113.45
           Connexion SSH RÉUSSIE pour 'admin' après 40 échecs depuis la même IP : compte probablement compromis

 CRITICAL  #2   WEB_SQL_INJECTION  [T1190]  source : 203.0.113.77
           Tentative d'injection SQL : 28 requête(s), dont 17 avec réponse 2xx — à vérifier en priorité
...
```

Un exemple de rapport est disponible dans [`examples/rapport-exemple.html`](examples/rapport-exemple.html).

## 📖 Utilisation

```bash
# Analyse d'un serveur réel (droits de lecture sur /var/log requis)
sudo logsentinel analyze --auth /var/log/auth.log --web /var/log/nginx/access.log

# Plusieurs fichiers, y compris les archives logrotate
logsentinel analyze --auth /var/log/auth.log --auth /var/log/auth.log.2.gz

# Uniquement les alertes graves, export JSON pour un SIEM
logsentinel analyze --web access.log --min-severity HIGH --json alertes.json

# Seuils personnalisés
logsentinel analyze --auth auth.log --config examples/config.json
```

| Option | Description |
|---|---|
| `--auth FICHIER` | Journal SSH (`auth.log`, `secure`), répétable, `.gz` accepté |
| `--web FICHIER` | Journal web Apache/Nginx, répétable, `.gz` accepté |
| `--year AAAA` | Année des logs syslog (le format ne la contient pas) |
| `--config JSON` | Fichier de seuils (voir `examples/config.json`) |
| `--json / --html FICHIER` | Exporter le rapport |
| `--min-severity NIVEAU` | Filtrer : `LOW`, `MEDIUM`, `HIGH`, `CRITICAL` |
| `--fail-on NIVEAU` | Code de sortie `2` si une alerte ≥ NIVEAU est trouvée |
| `--no-color`, `-q` | Sortie sans couleur / silencieuse |

### Surveillance automatique (cron)

```cron
# Toutes les heures : rapport HTML + e-mail si alerte HIGH ou plus
0 * * * * logsentinel analyze --auth /var/log/auth.log -q --fail-on HIGH --html /var/www/rapport.html || echo "Alerte LogSentinel" | mail -s "Intrusion possible" admin@example.org
```

## ⚙️ Configuration

| Clé | Défaut | Rôle |
|---|---|---|
| `ssh_bruteforce_threshold` | 5 | Échecs déclenchant une alerte de force brute |
| `ssh_bruteforce_window` | 300 | Fenêtre glissante (s) |
| `ssh_spray_distinct_users` | 5 | Comptes distincts pour le password spraying |
| `compromise_lookback` | 3600 | Période (s) examinée avant une connexion réussie |
| `web_404_threshold` / `web_404_window` | 20 / 60 | Énumération de contenu |
| `web_flood_threshold` / `web_flood_window` | 300 / 60 | Flood applicatif |
| `evidence_limit` | 5 | Lignes de preuve conservées par alerte |
| `whitelist` | `[]` | IP ou réseaux CIDR ignorés |

## 🏗️ Architecture

```
logsentinel/
├── logsentinel/
│   ├── cli.py          # Interface en ligne de commande
│   ├── parsers.py      # Lecture et normalisation des journaux
│   ├── signatures.py   # Signatures d'attaques web (regex)
│   ├── detectors.py    # Moteur de détection (fenêtres glissantes, corrélation)
│   ├── reporters.py    # Rapports console / JSON / HTML
│   ├── config.py       # Seuils et whitelist
│   ├── models.py       # Événements, alertes, sévérités
│   └── samples.py      # Générateur de journaux de démonstration
├── tests/              # 31 tests unitaires et d'intégration
├── examples/           # Journaux, configuration et rapports d'exemple
└── .github/workflows/  # Intégration continue (Python 3.9 → 3.13)
```

Le flux est volontairement simple : **journaux → parsers → événements normalisés → détecteurs → alertes → rapports**. Ajouter une règle revient à écrire une fonction `detect_xxx(events, cfg) -> list[Alert]` et à l'inscrire dans `AUTH_DETECTORS` ou `WEB_DETECTORS`.

## ✅ Tests

```bash
pip install -e ".[dev]"
pytest -v            # ou, sans dépendance : python -m unittest discover -s tests
```

## ⚠️ Limites

- Les signatures reposent sur des expressions régulières : elles ne remplacent pas un WAF ni un IDS réseau (Suricata, Zeek).
- Le corps des requêtes `POST` n'apparaît pas dans les access logs et n'est donc pas analysé.
- Une alerte signale une **tentative** ; une réponse `2xx` n'est pas une preuve de succès, mais un indicateur à vérifier.

## 🗺️ Pistes d'évolution

- [ ] Géolocalisation et réputation des IP (AbuseIPDB, GeoLite2)
- [ ] Mode temps réel (`--follow`, à la manière de `tail -f`)
- [ ] Export au format Sigma ou envoi vers Elasticsearch / Splunk
- [ ] Génération automatique de règles `iptables` / `fail2ban`
- [ ] Support des journaux Windows (EVTX) et `journalctl`

## 📜 Licence et usage

Distribué sous licence MIT (voir [LICENSE](LICENSE)). N'analysez que des journaux de systèmes que vous administrez ou pour lesquels vous disposez d'une autorisation. Les adresses IP des exemples appartiennent aux plages réservées à la documentation (RFC 5737).
