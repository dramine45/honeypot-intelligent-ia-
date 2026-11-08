# 🍯 Honeypot Intelligent — ELK Stack + IA Hybride

> **Déploiement d'un Honeypot Intelligent avec Analyse Automatisée via la Stack ELK et Techniques d'Intelligence Artificielle**

Projet réalisé à l'**INPT** (Institut National des Postes et Télécommunications) en partenariat avec l'**ANRT**, dans le cadre d'un PFA (Projet de Fin d'Année) — Fév.–Avr. 2026.

**Réalisé par :** DIOURI Amine · SAHBANI Ismail · BOUTFOUST Ayman  
**Encadré par :** Mme. CHARIFA Hanin · Mme. ELKANDOUSSI Asmaa

---

## 📋 Table des matières

- [Contexte et Problématique](#contexte-et-problématique)
- [Objectif](#objectif)
- [Architecture globale](#architecture-globale)
- [Technologies utilisées](#technologies-utilisées)
- [Moteur IA Hybride](#moteur-ia-hybride)
- [Pipeline de données](#pipeline-de-données)
- [Déploiement de l'infrastructure](#déploiement-de-linfrastructure)
- [Tests et Résultats](#tests-et-résultats)
- [Cas d'étude : Supériorité de l'IA](#cas-détude--supériorité-de-lia)
- [Inclusions / Exclusions](#inclusions--exclusions)
- [Améliorations possibles](#améliorations-possibles)
- [Bibliographie](#bibliographie)

---

## 🎯 Contexte et Problématique

Les applications web font face à une multiplication des cyberattaques (injections SQL, XSS, Path Traversal, Brute Force, etc.). Les systèmes de détection classiques fondés sur des **signatures statiques** présentent trois limites majeures :

- **Contournement par variantes** : un attaquant qui encode une URL ou modifie légèrement un payload passe à travers des règles fixes.
- **Faux positifs élevés** : sans contexte comportemental, les règles de signature génèrent un bruit important qui surcharge les analystes SOC.
- **Absence de visibilité sur les attaques zero-day** : non détectables par les systèmes à base de signatures.

**Question centrale :** Comment détecter de manière proactive des attaques web sophistiquées et *Zero-Day* sans noyer les analystes sous les faux positifs ?

---

## 🎯 Objectif

Concevoir et déployer un système de détection intelligent combinant :

1. Un **capteur actif (Honeypot)** — OWASP Juice Shop
2. Une **infrastructure de centralisation** — Stack ELK (Elasticsearch, Logstash, Kibana)
3. Une **couche d'IA hybride** — Isolation Forest + Random Forest (via FastAPI)

---

## 🏗️ Architecture globale

```
                        ┌─────────────────────────────────────────┐
                        │             Zone LAN (DMZ)              │
                        │                                         │
Attaquants  ──►  pfSense ──►  V-switch ──►  Honeypot             │
(OPT1)       (Firewall)              │     (OWASP Juice Shop)     │
192.168.242.x  10.10.10.1            │      10.10.10.6            │
                                     │                            │
                                     ├──►  Stack ELK              │
                                     │     10.10.10.7             │
                                     │     (Logstash, ES, Kibana) │
                                     │                            │
                                     └──►  Moteur IA              │
                                           10.10.10.20            │
                                           (Python, FastAPI)      │
                        └─────────────────────────────────────────┘
```

### Inventaire des machines virtuelles

| Machine | IP | Services déployés |
|---|---|---|
| Honeypot + Filebeat | `10.10.10.6` | OWASP Juice Shop (port 3000/80), Filebeat 8.x |
| ELK Stack | `10.10.10.7` | Elasticsearch :9200, Logstash :5044, Kibana :5601 |
| Serveur IA/ML | `10.10.10.20` | Python 3.10, FastAPI :8000, scikit-learn, cron 5 min |
| pfSense | `10.10.10.1` | Firewall, NAT WAN/DMZ/LAN |

---

## 🛠️ Technologies utilisées

| Couche | Outil / Technologie |
|---|---|
| Honeypot | OWASP Juice Shop (Node.js, OWASP Top 10 complet) |
| Collecte de logs | Filebeat 8.x |
| Pipeline & enrichissement | Logstash (filtres Grok, décodage URL) |
| Stockage & indexation | Elasticsearch (index `soc-juiceshop-*`, `detections-ia`) |
| Visualisation & alerting | Kibana (Discover, Security Rules, Dashboards) |
| Moteur IA | Python 3.10, scikit-learn, joblib |
| API d'inférence | FastAPI (routes `/predict`, `/poll`, `/test`) |
| Firewall & réseau | pfSense 2.7, VirtualBox, Virtual Switch |
| Télémétrie réseau | Syslog BSD (RFC 3164) → Logstash UDP 5140 |

### Pourquoi OWASP Juice Shop ?

OWASP Juice Shop a été retenu face à Tanner (ex-Glastopf) et DVWA pour :
- **Couverture OWASP Top 10 complète** (10 catégories d'attaques)
- **Logs JSON natifs** directement exploitables par Filebeat sans parseur custom
- **Interface crédible** (boutique e-commerce réaliste)
- **Maintenance active** par l'OWASP officiel

---

## 🤖 Moteur IA Hybride

Le moteur combine deux modèles complémentaires pour une couverture maximale.

### Isolation Forest — Détection comportementale (non supervisé)

Détecte ce qui s'écarte de la norme statistique, sans signature préexistante.

| Paramètre | Valeur | Signification |
|---|---|---|
| `n_estimators` | 100 | Nombre d'arbres d'isolation |
| `contamination` | 0.2 | 20 % du trafic estimé anormal |
| `n_features` | 5 | Features comportementales |

**5 features extraites :**
1. Longueur de l'URL
2. Code HTTP de la réponse
3. Type de méthode HTTP (GET ou non-GET)
4. Présence de caractères suspects dans l'URL
5. Présence de mots sensibles (`admin`, `etc`, `passwd`…)

✅ **Avantage :** Détection d'attaques inconnues (zero-day), ne nécessite aucune donnée labellisée.

---

### Random Forest — Détection par signatures (supervisé)

Classifie chaque requête en `BENIGN` ou `WEB_ATTACK` avec un score de confiance.

| Paramètre | Valeur | Signification |
|---|---|---|
| `n_estimators` | 100 | Nombre d'arbres de décision |
| `n_features` | 8 | Features orientées signatures |
| `classes` | `BENIGN`, `WEB_ATTACK` | Classes de sortie |

**8 features extraites :**
1. Longueur de l'URL
2. Nombre de mots critiques (`select`, `union`, `script`, `alert`, `whoami`…)
3. Présence d'encodage `%` dans l'URL
4. Code HTTP de la réponse
5. Indicateur d'erreur HTTP (4xx/5xx)
6. Présence de codes 401 ou 403
7. Méthode HTTP (GET ou non-GET)
8. Nombre de séquences de traversal (`../` ou `..%2f`)

> **Note :** Le modèle a été entraîné sur des données custom issues de l'environnement honeypot. Le dataset public CICIDS2017 a été écarté (faux positifs importants dûs à la divergence de longueurs d'URLs).

✅ **Avantage :** Précision élevée sur les patterns connus (SQLi, XSS, Path Traversal), score de confiance interprétable.

---

### Complémentarité IF + RF

```
Une alerte est déclenchée dès qu'au moins l'un des deux modèles signale une activité suspecte.

  if_score < 0          →  ALERTE [IF]       (anomalie comportementale)
  rf_label = WEB_ATTACK →  ALERTE [RF]       (signature connue)
  les deux              →  ALERTE [RF+IF]    (suspicion maximale)
```

---

## 🔄 Pipeline de données

```
OWASP Juice Shop (access.log)
        │
        ▼ Filebeat (JSON)
   Logstash :5044
   (Grok filters, URL decode, enrichissement)
        │
        ▼
  Elasticsearch
  index: soc-juiceshop-YYYY.MM.DD
        │
        ▼ GET _search (polling continu)
  app_hybrid.py (FastAPI + IF + RF)
        │
        ▼ POST (alertes)
  index: detections-ia
        │
        ▼
  Kibana Security Alerts
```

Le fichier central `app_hybrid.py` orchestre :
- Chargement des 4 modèles sérialisés au démarrage (`isolation_forest.joblib`, `random_forest.joblib`, `rf_scaler.joblib`, `rf_label_encoder.joblib`)
- Extraction des champs via `get_url()`, `get_status()`, `get_method()`, `get_ip()`
- Prédiction par les deux modèles et décision finale
- Enregistrement dans `detections-ia` et affichage console temps réel

---

## 🚀 Déploiement de l'infrastructure

### Configuration pfSense

- **Port Forwarding (NAT) :** Tout trafic TCP entrant sur OPT1 → port 3000 → redirigé vers le honeypot `10.10.10.6:3000`
- **Remote Logging (Syslog) :** Logs système + firewall envoyés vers `10.10.10.7` via UDP 5140, format BSD (RFC 3164) pour compatibilité Logstash

### Arborescence du projet IA

```
detection_ia/
├── app_hybrid.py          # Moteur principal FastAPI (IF + RF)
├── models/
│   ├── isolation_forest.joblib
│   ├── random_forest.joblib
│   ├── rf_scaler.joblib
│   └── rf_label_encoder.joblib
└── scripts/
    └── train_models.py    # Entraînement des modèles
```

### Commandes de vérification Elasticsearch

```bash
# Volume total de logs collectés
curl -u amine:admin123 "http://10.10.10.7:9200/soc-juiceshop-*/_count"
# → {"count":11392, "_shards":{"total":4,"successful":4,...}}

# 5 dernières détections IA
curl -s -u amine:admin123 \
  "http://10.10.10.7:9200/detections-ia/_search?size=5&sort=@timestamp:desc" \
  | python3 -m json.tool

# Liste des indices avec statistiques
curl -u amine:admin123 \
  "http://10.10.10.7:9200/_cat/indices?v&h=index,docs.count,store.size,health&s=index"
```

---

## 📊 Tests et Résultats

### Script de test (`test_soc_rules.sh`)

Un script Bash simule **8 catégories d'attaques** depuis la machine honeypot (`localhost:3000`) :

| # | Catégorie | Exemples de payloads |
|---|---|---|
| 1 | **XSS** | `<script>alert(1)</script>`, `onerror`, `javascript:alert(0)` |
| 2 | **Brute Force Login** | 10 POST vers `/rest/user/login` avec mots de passe aléatoires |
| 3 | **SQL Injection** | `' OR 1=1--`, `UNION SELECT * FROM users--`, `'; DROP TABLE users;--` |
| 4 | **Path Traversal** | `/../../../etc/passwd`, `/etc/shadow`, version URL-encodée |
| 5 | **Sensitive File Access** | `/ftp/`, `/ftp/package.json`, `/ftp/secret.txt` |
| 6 | **Admin Panel Access** | `/#/administration`, `/rest/admin/application-version`, `/api/admin` |
| 7 | **Command Injection** | `?q=apple;ls`, `?q=test\|cat /etc/passwd`, `?q=test;whoami` |
| 8 | **Scanner Detection** | User-Agents : `sqlmap/1.7`, `Nikto/2.1.6`, `zgrab/0.1` |

### Résultats obtenus

```
✅ 8/8 catégories d'attaques simulées avec succès
📊 11 392 logs HTTP ingérés dans Elasticsearch
🤖 149 détections IA générées (index detections-ia)
🚨 231 alertes Kibana Security déclenchées
```

### Règles SOC Kibana (8 règles personnalisées)

| Règle | Sévérité | Risk Score | Statut |
|---|---|---|---|
| SOC - Command Injection Attempt | 🔴 Critical | 99 | ✅ OK |
| SOC - SQL Injection | 🔴 Critical | 99 | ✅ OK |
| SOC - XSS Attack Detected | 🟠 High | 73 | ✅ OK |
| SOC - Path Traversal Attempt | 🟠 High | 73 | ✅ OK |
| SOC - Admin Panel Access Attempt | 🟠 High | 73 | ✅ OK |
| SOC - Brute Force Login | 🟠 High | 73 | ✅ OK |
| SOC - Sensitive File Access | 🟡 Medium | 47 | ✅ OK |
| SOC - Automated Scanner Detected | 🟡 Medium | 47 | ✅ OK |

### Distribution des 231 alertes Kibana

- **Brute Force Login :** 141 alertes
- **Command Injection :** 36 alertes (Critical)
- **Path Traversal :** 24 alertes
- **Admin Panel Access :** 16 alertes
- Source exclusive : 100 % du host honeypot → LAN correctement isolé

---

## 🔬 Cas d'étude : Supériorité de l'IA

### Scénario : Contournement XSS

**Payload injectée :**
```
?q=<body onbeforeprint=console.log(1)>
```
Cette variante n'utilise **pas** la balise `<script>` habituelle.

**Résultat côté SIEM classique (règle statique Kibana) :**
```
Règle : message: *script* OR url.path: *script*
→ ❌ Aucune alerte générée — l'attaque passe inaperçue
```

**Résultat côté moteur IA :**
```
ALERTE [RF] | ? | RF=WEB_ATTACK (100.0%) | IF= 0.0513
/search?q=%3Cbody%20onbeforeprint=console.log(1)%3E
→ ✅ Détecté avec 100% de confiance par le Random Forest
```

Le Random Forest analyse les **caractéristiques globales de l'URL** (structure, encodage, longueur, présence de séquences anormales) plutôt qu'un simple mot-clé — rendant le contournement très difficile.

---

## ✅ Inclusions / ❌ Exclusions

### Inclusions
- Collecte et analyse de logs en temps réel (Stack ELK)
- Déploiement d'un leurre web crédible (OWASP Juice Shop)
- Détection par IA Hybride (Isolation Forest + Random Forest)
- Pare-feu et isolation réseau (pfSense)

### Exclusions
- ❌ Bloquage actif du trafic (pas de fonction IPS)
- ❌ Déploiement sur un LAN utilisateur réel (réseau totalement isolé)

---

## 🔭 Améliorations possibles

1. **Deep Learning :** Intégration d'un module **CNN-LSTM** pour détecter des séquences d'attaques multi-étapes sur la durée.
2. **Réseau :** Exploitation totale des logs bruts du pare-feu pfSense par le modèle IA (corrélation réseau + applicatif).
3. **Automatisation SOC :** Mise en place d'un assistant IA pour aider les analystes à trier les alertes automatiquement.

---

## 📚 Bibliographie

- [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/)
- [Documentation ELK Stack](https://www.elastic.co/fr/elk-stack)
- [Scikit-learn — Random Forest](https://scikit-learn.org/stable/modules/ensemble.html)
- [Scikit-learn — Isolation Forest](https://scikit-learn.org/stable/modules/outlier_detection.html)
- [Documentation Python 3](https://docs.python.org/3/)
- [Documentation FastAPI](https://fastapi.tiangolo.com/)

---

*INPT — Institut National des Postes et Télécommunications | ANRT — Agence Nationale de Réglementation des Télécommunications*
