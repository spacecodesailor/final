# 📝 MÉMO TECHNIQUE SÉCURISÉ : PIPELINE CI/CD & DOCKER SWARM
## Projet : Sfeirschool 2018 (Multi-job Back & Front)

---

### 🛡️ 1. CONFIGURATION SÉCURISÉ DU RUNNER (Option 1)
Pour éviter la faille de sécurité du `chmod 666 /var/run/docker.sock`, le runner est configuré pour s'exécuter en tant que **service système Linux**. Il utilise les permissions d'origine (`660`) du socket en s'appuyant strictement sur le groupe `docker`.

Exécutez ces commandes dans le terminal de votre machine pour appliquer cette configuration :

```bash
# 1. Rétablir les permissions sécurisées par défaut sur le socket Docker
sudo chmod 660 /var/run/docker.sock

# 2. S'assurer que l'utilisateur 'student' appartient bien au groupe docker
sudo usermod -aG docker student

# 3. Entrer dans le dossier du runner et installer le service système officiel
cd ~/actions-runner
sudo ./svc.sh install student

# 4. Démarrer le service du runner en arrière-plan
sudo ./svc.sh start
```
*Le runner tourne désormais en tâche de fond de manière isolée, résiste aux redémarrages de la machine, et possède les droits Docker requis sans compromettre la sécurité globale du système.*

---

### 🏗️ 2. L'ARCHITECTURE DU PIPELINE (GitHub Actions)
Le fichier `.github/workflows/ci-cd.yml` gère l'automatisation complète en deux jobs séquentiels :

#### A. Le Job de CI (`build-and-test`) — Cloud GitHub
* **Stratégie de Matrice (`matrix`)** : Un seul bloc de code YAML s'exécute en parallèle pour les composants `[back, front]`, éliminant la duplication de code.
* **Le Short SHA Git** : Extraction automatique des 7 premiers caractères du commit pour taguer de manière unique et traçable les images Docker (`sfeir-back:sha_court`) et les Releases GitHub.

#### B. Le Job de CD (`deploy`) — Votre Machine (`runs-on: self-hosted`)
* **Déploiement Synchrone** : Utilisation de l'option `--detach=false` dans la commande `docker stack deploy` pour forcer Swarm à stabiliser la création des réseaux avant d'instancier les services, supprimant les erreurs de synchronisation (*race conditions*).

---

### 🐳 3. TOPOLOGIE DE L'ORCHESTRATION (Docker Swarm)
Le fichier `docker-compose.yaml` est configuré pour déployer l'architecture haute disponibilité suivante :

* **Services et Dimensionnement** :
  * `webui` : Interface graphique de visualisation (Exposée sur le port `8000`).
  * `front` : Composant Frontend (Déployé en **25 réplicas**).
  * `back` : Composant Backend (Déployé en **50 réplicas**).
  * `db` : Base de données CouchDB (Données persistées via un volume).

* **Gestion IPAM & Résolution du "Pool Overlap"** :
  Pour empêcher Docker Swarm d'entrer en conflit avec le réseau interne de la machine hôte lors du déploiement massif des 75 conteneurs, des sous-réseaux isolés de classe `172.x.x.x` sont explicitement définis :
  * `db_net` : Plage d'adresses `172.28.0.0/16`
  * `web_net` : Plage d'adresses `172.29.0.0/16`

---

### 🛠️ 4. GUIDE DE SURVIE CLI (Commandes de diagnostic)

#### Suivi et Supervision du Cluster
```bash
# Vérifier l'état des services de votre stack et le décompte des réplicas actifs
docker service ls

# Suivre la montée en charge et le démarrage des conteneurs en temps réel
watch docker service ls

# Inspecter l'état individuel de chaque tâche/conteneur de la stack
docker stack ps final-pipeline
```

#### Analyse des Logs et Gestion du Service Runner
```bash
# Consulter les logs de l'application Backend en direct (Option -f pour suivre le flux)
docker service logs -f final-pipeline_back

# Vérifier le statut du service système de votre runner GitHub
cd ~/actions-runner
sudo ./svc.sh status
```

added cache