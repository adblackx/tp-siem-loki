# 🛡️ TP : Opérations SOC & Cyberdéfense Active

Bienvenue dans le SOC (**Security Operations Center**).
Votre mission : Construire un outil de surveillance (SIEM), détecter des attaques en temps réel, analyser les preuves réseaux, et neutraliser les menaces.

## 🏗️ L'Architecture du Lab

Nous utilisons une architecture moderne et légère ("Cloud Native") :

* **La Cible (Zone DMZ)** : Un serveur Web Nginx et un serveur SSH.
* **L'Attaquant** : Un conteneur Kali Linux qui lance des attaques cycliques (Bruteforce, SQLi, DoS, Scan).
* **Le SIEM (Surveillance)** :
* **Promtail** : L'agent qui lit les logs.
* **Loki** : La base de données de logs.
* **Grafana** : L'interface visuelle (Tableaux de bord & Alertes).


* **Le Poste Analyste** : **Jupyter** (Python) pour l'investigation avancée et la riposte.

---

## 🚀 Démarrage Rapide

### 1. Lancer l'infrastructure

Ouvrez votre terminal dans le dossier du projet et lancez :

```bash
docker compose up -d

```

### 2. Accéder aux Outils

Une fois les conteneurs lancés (attendre ~30 secondes), ouvrez votre navigateur :

| Outil | URL | Identifiants | Usage |
| --- | --- | --- | --- |
| **Grafana** | `http://localhost:3000` | User: `admin` <br>

<br> Pass: `admin` | Visualisation & Alertes |
| **Jupyter** | `http://localhost:8888` | Token: `securetoken123` | Analyse de code & Riposte |

---

## 📘 Guide de Configuration Grafana (Pas à pas)

Grafana est vide au démarrage. Vous devez le configurer pour voir les attaques. Suivez ce guide méticuleusement.

### Étape 1 : Connecter la Source de Données (Loki)

Grafana a besoin de savoir où sont stockés les logs.

1. Connectez-vous à Grafana (`admin` / `admin`). Passez l'écran de changement de mot de passe (cliquez sur "Skip").
2. Dans le menu de gauche, allez sur **Connections** (l'icône de prise) > **Data Sources**.
3. Cliquez sur le bouton bleu **+ Add new data source**.
4. Cherchez et sélectionnez **Loki**.
5. **Configuration Critique :**
* Dans le champ **URL**, saisissez exactement : `http://loki:3100`
* *Pourquoi ?* `loki` est le nom du conteneur dans le réseau Docker interne.


6. Descendez tout en bas et cliquez sur **Save & Test**.
* ✅ *Succès :* Un bandeau vert "Data source successfully connected" apparaît.



### Étape 2 : Comprendre l'interface "Explore" vs "Dashboard"

* **Explore (La Boussole) :** C'est votre bac à sable. Cliquez sur l'icône "Explore" à gauche. Sélectionnez "Loki" en haut. C'est ici qu'on tape des requêtes manuelles pour chercher une preuve précise.
* **Dashboard (Le Tableau de Bord) :** C'est ici qu'on fige les graphiques pour la surveillance continue.

### Étape 3 : Créer le Dashboard "SOC Overview"

Nous allons créer 3 panneaux pour détecter 3 types d'attaques.

1. Allez dans **Dashboards** (menu gauche) > **New Dashboard** > **+ Add visualization**.
2. Sélectionnez **Loki** comme source.
3. En bas de l'écran, passez l'éditeur en mode **Code** (cliquez sur le bouton `Builder` pour qu'il devienne `Code`).

#### 📊 Panneau A : Détection Bruteforce SSH

L'attaquant essaie de deviner le mot de passe root.

* **Requête LogQL :** Copiez ceci dans la zone de code :
```logql
count_over_time({container="victim_server"} |= "Failed password" [1m])

```


* **Configuration Visuelle (à droite) :**
* **Title :** `🚨 SSH Bruteforce`
* **Graph styles > Line color :** Mettez du Rouge.
* **Graph styles > Fill opacity :** 50%.


* Cliquez sur **Apply** (en haut à droite).

#### 📊 Panneau B : Détection Fuzzing Web (404)

L'attaquant cherche des pages cachées (`admin.php`, `backup.sql`), générant des erreurs 404.

* Cliquez sur l'icône **Add Panel** (en haut du dashboard).
* **Requête LogQL :**
```logql
sum(count_over_time({container="web_server"} | json | status="404" [1m]))

```


*Note : On utilise `| json` car Nginx est configuré pour envoyer des logs structurés.*
* **Configuration Visuelle :**
* **Title :** `🔍 Web Fuzzing (404 Errors)`
* **Visualisation (en haut à droite) :** Changez "Time series" par **Bar gauge**.
* **Thresholds (Seuils) :** Réglez le rouge à partir de **20**.


* Cliquez sur **Apply**.

#### 📊 Panneau C : Détection de Scanner (Signature Nmap)

L'attaquant utilise l'outil Nmap, qui laisse sa signature dans le "User-Agent".

* Ajoutez un nouveau panneau.
* **Requête LogQL :**
```logql
sum(count_over_time({container="web_server"} | json | http_user_agent =~ ".*Nmap.*" [1m]))

```


* **Configuration Visuelle :**
* **Title :** `🤖 Nmap Scan Detected`
* **Visualisation :** Changez pour **Stat**.
* **Color mode :** Choisissez "Background".


* Cliquez sur **Apply**, puis sur l'icône **Save** (disquette) pour sauvegarder votre Dashboard.



---





### Étape 4. Configurer les Alertes (Le cerveau du SIEM)

C'est ici que l'on passe de la simple vue à la surveillance active.

1. Dans l'édition d'un panneau, allez dans l'onglet **Alert**.
2. Cliquez sur **Create alert rule from this panel**.
3. **Condition :** Réglez le seuil. Par exemple : `IS ABOVE 15`.
4. **Notifications :** Allez dans **Alerting** > **Contact points** pour lier Grafana à un webhook (Discord, Slack, ou même un script Python personnalisé).



## 🕵️‍♂️ Vos Missions

Maintenant que le SIEM est prêt, vous êtes l'analyste en poste.

### Mission 1 : Analyse de Logs (Threat Hunting)

*Outil : Jupyter Notebook `SOC_Analyst_Training.ipynb*`

1. Ouvrez le Notebook dans Jupyter.
2. Exécutez la **Cellule 2**.
3. **Objectif :** Trouvez l'adresse IP exacte de l'attaquant qui effectue des injections SQL (`' OR 1=1`). Le SIEM (Grafana) vous dit *quand* ça arrive, Jupyter vous dit *qui* et *comment*.

### Mission 2 : Analyse Réseau (Forensics)

*Outil : Terminal & Jupyter (Scapy)*

Certaines attaques (comme le Déni de Service "Slowloris") sont peu visibles dans les logs.

1. Dans un terminal, vérifiez que l'attaquant est actif :
```bash
docker logs -f attacker_bot
# Attendez de voir : [PHASE 3] Slowloris DoS... ou [PHASE 4] Nmap Scan...

```


2. Lancez la capture sur la victime :
```bash
#docker exec -it web_server tcpdump -i any -n -w /tmp/evidence/capture.pcap
docker exec -it web_server tcpdump -i any -w /tmp/evidence/capture.pcap

```

3. **Laissez tourner pendant 30 secondes** (pour capturer un cycle d'attaque complet).
4. Arrêtez la capture avec `CTRL + C`.
5. Retournez dans Jupyter (Cellule 3). Le fichier `.pcap` est automatiquement partagé.
6. Ouvrez le Notebook Jupyter pour analyser le fichier `.pcap`.
7. Exécutez l'analyse Scapy. Vous devriez voir un **SYN Flood** (beaucoup de demandes de connexion incomplètes).
* *Vérification :* Vous devez voir le message `X packets captured` (où X > 0).




### Mission 3 : Défense Active (La Riposte)

*Outil : Jupyter (Active Response)*

Il est temps de stopper l'attaque.

1. Dans Jupyter (Cellule 4), utilisez le script Python fourni pour interagir avec le Pare-feu (Iptables) de la victime.
2. Bloquez l'IP de l'attaquant.
```python
block_ip_firewall("172.xx.0.xx") # Remplacez par l'IP trouvée en Mission 1

```


3. Retournez sur **Grafana** : Vérifiez que toutes les courbes retombent à zéro. C'est la preuve de votre succès.

---

## 📚 Annexe Technique

### Pourquoi ces requêtes LogQL ?

* `{container="..."}` : Filtre les logs par source.
* `|= "texte"` : Cherche si la ligne contient le texte exact (Recherche simple).
* `| json` : Transforme la ligne de log en objet manipulable (permet de filtrer par `status` ou `user_agent`).
* `[1m]` : Calcule le débit par minute.

### L'Attaquant

Le conteneur `attacker` exécute un script cyclique :

1. **Reconnaissance :** Nmap.
2. **Web :** Injection SQL & Path Traversal.
3. **DoS :** Slowloris.
4. **Bruteforce :** Hydra (SSH).