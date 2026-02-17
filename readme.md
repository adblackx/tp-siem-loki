Voici un fichier **README.md** complet et structuré, conçu pour documenter votre TP. Il récapitule l'architecture, la configuration et les méthodologies de détection que nous avons mises en place.

---

# TP : Conception d'un SIEM Lightweight (Loki / Grafana / Jupyter)

## 📋 Présentation

Ce TP a pour objectif de concevoir un système de gestion des événements et des informations de sécurité (**SIEM**) léger et moderne. Contrairement aux solutions lourdes basées sur Elasticsearch (ELK), nous utilisons ici la stack **PLG** (Promtail, Loki, Grafana) pour monitorer des conteneurs Docker en temps réel.

## 🏗️ Architecture Technique

L'infrastructure est entièrement conteneurisée et orchestrée via `docker-compose`.

* **Victimes :** * `web_server` (Nginx) : Configuré pour générer des logs au format **JSON** pour une analyse structurée.
* `victim_server` (Alpine SSH) : Serveur Linux standard simulant une cible d'attaque par force brute.


* **Collecte & Stockage :**
* **Promtail** : Agent qui "aspire" les logs du socket Docker (`/var/run/docker.sock`).
* **Loki** : Base de données optimisée pour le stockage des logs (le "cerveau").


* **Analyse & Visualisation :**
* **Grafana** : Interface de visualisation (Dashboards) et moteur d'alerting.
* **JupyterHub** : Interface de programmation Python pour le *Threat Hunting* automatisé.


* **Attaquant :**
* `attacker_bot` (Kali Linux) : Génère des attaques cycliques (Hydra pour le SSH, Curl pour le Fuzzing Web, Nmap pour le scan réseau).



---

## 🛠️ Configuration des Services

### Logging JSON (Nginx)

Pour faciliter la détection, nous avons surchargé la configuration par défaut de Nginx pour produire des logs structurés. Cela permet d'extraire directement des champs comme `status` ou `http_user_agent` sans utiliser de RegEx complexes.

```nginx
log_format json_analytics escape=json '{'
    '"status": "$status", '
    '"http_user_agent": "$http_user_agent", '
    '"request_uri": "$request_uri"'
'}';

```

### Automatisation de l'Attaque

L'attaquant utilise des variables Bash (`$$i` dans Docker Compose) pour simuler un comportement agressif en continu :

* **SSH** : Tentatives de connexion root en boucle.
* **Web** : Requêtes sur des pages inexistantes (404).
* **Reconnaissance** : Injection de signatures Nmap dans les headers HTTP.

---

## 🔍 Méthodes d'Analyse (LogQL)

Dans Grafana, nous avons implémenté trois niveaux de détection utilisant le langage **LogQL** :

### 1. Détection de Bruteforce SSH (Analyse Textuelle)

On compte les échecs de mot de passe sur le serveur de la victime :

```logql
count_over_time({container="victim_server"} |= "Failed password" [1m])

```

### 2. Détection de Fuzzing Web (Analyse JSON)

On utilise le parser JSON pour identifier les pics d'erreurs 404 :

```logql
sum(count_over_time({container="web_server"} | json | status="404" [1m]))

```

### 3. Détection de Scanner Nmap (Signature Based)

On recherche la signature spécifique de l'outil Nmap dans les User-Agents :

```logql
sum(count_over_time({container="web_server"} | json | http_user_agent =~ ".*Nmap.*" [1m]))

```

---

## 🚦 Monitoring vs Alerting

C'est le cœur du métier d'analyste SOC (Security Operations Center) :

* **Explore** : Utilisé pour l'investigation manuelle (chercher un log spécifique suite à un incident).
* **Dashboard** : Utilisé pour la surveillance visuelle des tendances.
* **Alerting** : Utilise les requêtes LogQL pour déclencher une notification automatique.

> **Exemple de règle d'alerte :** Si `status="404"` > 20 sur une période de 1 minute, l'alerte passe en état **Firing** (Rouge), signalant une attaque de découverte de répertoire.

---

## 🚀 Utilisation

1. Lancer l'infrastructure : `docker compose up -d`
2. Accéder à Grafana : `http://localhost:3000` (admin/admin)
3. Accéder à Jupyter : `http://localhost:8888` (token: securetoken123)
4. Observer les pics d'attaques sur le Dashboard "SOC Overview".

---

**TP réalisé dans le cadre d'une initiation à la Cybersécurité Cloud-Native.**

---

Souhaites-tu que j'ajoute une section spécifique sur le code Python que nous avons écrit dans Jupyter pour le calcul des seuils ?