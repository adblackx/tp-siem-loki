Voici un guide spécifique pour dompter la configuration de Grafana, qui peut effectivement être intimidante au début, suivi d'idées pour transformer ce TP en un projet de plusieurs jours.

---

# 📘 Guide de Configuration Grafana (SIEM Edition)

La configuration de Grafana pour un SIEM repose sur trois piliers : la **Source de données**, la **Visualisation** et l'**Alerte**.

### 1. Connecter Loki (La Source)

Avant de créer des graphiques, Grafana doit savoir où chercher les logs.

1. Connectez-vous sur `http://localhost:3000` (Login/Pass: `admin`).
2. Allez dans **Connections** > **Data Sources**.
3. Cliquez sur **Add data source** et choisissez **Loki**.
4. Dans le champ **HTTP -> URL**, saisissez : `http://loki:3100`.
* *Note : On utilise `loki` car c'est le nom du service dans le réseau Docker.*


5. Descendez tout en bas et cliquez sur **Save & Test**. Un bandeau vert doit confirmer la connexion.

### 2. Créer un Dashboard "SOC"

Un Dashboard est une collection de panneaux (Panels). Pour ce TP, nous allons créer trois visualisations distinctes pour surveiller les différentes phases de l'attaque.

1. Allez dans **Dashboards** > **New** > **New Dashboard**.
2. Cliquez sur **Add Visualization**. Sélectionnez **Loki**.
3. **Le sélecteur de logs :** En bas, passez en mode **Code** (au lieu de Builder).
4. Saisissez votre requête LogQL : `sum(count_over_time({container="web_server"} | json | status="404" [1m]))`.
5. **Type de vue :** Dans le menu de droite, changez "Time series" par "Bar Gauge" ou "Stat" pour varier les plaisirs.
6. Donnez un titre (ex: "Détection Fuzzing Web") et cliquez sur **Apply**.


#### 📊 Panneau A : Tentatives de Force Brute (SSH)

* **Requête LogQL :** 
```logql
count_over_time({container="victim_server"} |= "Failed password" [1m])
```
* **Titre du Panel :** `🚨 Auth Security: SSH Brute Force Attempts`
* **Type de vue :** **Time series** (pour voir l'évolution des pics dans le temps).
* **Couleur :** Rouge (dans les options *Graph styles* > *Line color*).

#### 📊 Panneau B : Reconnaissance & Fuzzing (Web 404)

* **Requête LogQL :** 
```logql
sum(count_over_time({container="web_server"} | json | status="404" [1m]))
```
* **Titre du Panel :** `🔍 Web Security: Directory Discovery (Fuzzing)`
* **Type de vue :** **Bar Gauge** (pour visualiser l'intensité de l'attaque sous forme de jauge).
* **Seuils (Thresholds) :** Configure le jaune à partir de 10 et le rouge à partir de 30.

#### 📊 Panneau C : Détection de Scanner (Nmap)

* **Requête LogQL :** 
```logql
sum(count_over_time({container="web_server"} | json | http_user_agent =~ ".*Nmap.*" [1m]))
```
* **Titre du Panel :** `🤖 Threat Intel: Nmap Scanner Signature Detected`
* **Type de vue :** **Stat** (affiche un gros chiffre impactant).
* **Color mode :** Sélectionne "Background" pour que tout le carré devienne bleu ou rouge dès qu'un scan est détecté.


### 3. Configurer les Alertes (Le cerveau du SIEM)

C'est ici que l'on passe de la simple vue à la surveillance active.

1. Dans l'édition d'un panneau, allez dans l'onglet **Alert**.
2. Cliquez sur **Create alert rule from this panel**.
3. **Condition :** Réglez le seuil. Par exemple : `IS ABOVE 15`.
4. **Notifications :** Allez dans **Alerting** > **Contact points** pour lier Grafana à un webhook (Discord, Slack, ou même un script Python personnalisé).

---

# 🚀 Idées pour prolonger le TP (Extension du projet)

Pour faire durer ce TP (par exemple sur 2 ou 3 séances de 4h), voici des modules complémentaires à ajouter :

### Module 1 : Threat Hunting avec Jupyter (Analyse Forensique)

Au lieu de simplement regarder Grafana, les étudiants doivent utiliser le conteneur **Jupyter** pour :

* Extraire les logs de Loki via l'API Python.
* Identifier l'adresse IP de l'attaquant (`remote_addr`).
* **Challenge :** Automatiser le blocage. Créer un script Python qui, s'il détecte une attaque, utilise l'API Docker pour couper le réseau du conteneur `attacker_bot`.

### Module 2 : Obfuscation et Évasion (Red Team vs Blue Team)

* **L'Attaque :** Demandez aux étudiants de modifier le script de l'attaquant pour qu'il soit "furtif" (ex: lancer 1 requête toutes les 30 secondes au lieu de 50 d'un coup).
* **La Défense :** Les étudiants doivent ajuster les fenêtres de temps dans Loki (`[5m]` au lieu de `[1m]`) et les seuils d'alerte pour détecter ces attaques lentes ("Low and Slow").

### Module 3 : Enrichissement des Logs (GeoIP)

* Ajoutez un service de GeoIP (comme `MaxMind`) pour transformer les adresses IP des logs en pays.
* **Objectif :** Créer une carte du monde dans Grafana (Geomap) montrant d'où viennent les attaques. Cela apprend à manipuler des pipelines de données plus complexes.

### Module 4 : Sécurisation de la Stack (Hardening)

* Par défaut, Loki et Grafana n'ont pas de TLS ou d'authentification forte entre eux.
* **Challenge :** Mettre en place un Reverse Proxy (Nginx ou Traefik) devant Grafana avec du HTTPS et configurer Loki pour n'accepter que les connexions venant de Promtail avec un token d'authentification.

### Module 5 : Analyse de nouveaux services

* Ajoutez une base de données **PostgreSQL** ou **MySQL** à l'infrastructure.
* Simulez une attaque par injection SQL (via l'attaquant).
* **Objectif :** Trouver quel mot-clé dans les logs de la base de données permet de détecter une injection SQL (ex: `SELECT * FROM`, `UNION SELECT`).

**Souhaitez-vous que je développe le code Python pour le "Module 1" (blocage automatique de l'attaquant) ?**