# **Chapitre 2 : Architecture Technique**

## **2.1. Vue d'Ensemble de l'Architecture**

L'architecture du projet "Augmented Analyst" a été conçue pour garantir la robustesse, la fiabilité et l'automatisation. Elle repose sur une séparation claire des responsabilités entre trois environnements distincts : l'environnement de développement local, la plateforme d'intégration continue (CI/CD) et l'environnement de production.

Le flux de travail standard est le suivant :
1.  Le développement et les tests initiaux sont effectués dans l'**Environnement Local**.
2.  Chaque modification validée et "pushée" vers le dépôt Git déclenche le processus dans l'**Environnement de CI/CD**, qui agit comme un portail de contrôle qualité.
3.  Si, et seulement si, la qualité est validée, le code est automatiquement déployé dans l'**Environnement de Production**, où il est accessible à l'utilisateur final.

Cette structure garantit qu'aucune modification ne peut atteindre la production sans avoir été testée et validée, assurant ainsi une stabilité maximale du service.

## **2.2. L'Application Principale (Backend)**

Le cœur logique de l'application est un backend développé en Python, orchestré par le micro-framework Flask.

### **2.2.1. Framework**

L'application est construite avec **Flask**. Le code suit le design pattern de l'**Application Factory** (via la fonction `create_app`). Cette approche consiste à encapsuler la création et la configuration de l'application dans une fonction, ce qui offre plusieurs avantages :
*   **Testabilité :** Permet de créer différentes instances de l'application pour différents contextes (production, test) avec des configurations distinctes.
*   **Organisation :** Évite les variables d'application globales et favorise une structure de code plus propre et plus modulaire.

### **2.2.2. Modèles et Base de Données**

L'interaction avec la base de données est gérée par l'ORM **SQLAlchemy**, via l'extension **Flask-SQLAlchemy**. Cette approche permet de manipuler les données en utilisant des objets Python (modèles) plutôt que d'écrire des requêtes SQL brutes.

Le schéma de la base de données relationnelle est composé de trois tables principales :
*   **`user` :** Stocke les informations d'authentification (un seul utilisateur pour cette application).
*   **`titres` :** Table de référence contenant la liste des actifs financiers du portefeuille (ex: ticker, nom de l'entreprise, données sur 52 semaines).
*   **`historique` :** Table la plus volumineuse, qui enregistre une "photographie" de chaque titre pour chaque jour d'importation. Elle contient les colonnes clés :
    *   `valeur` : Le prix brut du titre, dans sa devise d'origine.
    *   `devise` : La devise du prix brut (ex: 'USD', 'CAD').
    *   `cad_value` : Une colonne pré-calculée contenant la valeur du prix unitaire, systématiquement convertie en CAD pour faciliter les agrégations.
    *   `quantite` : La quantité détenue pour ce relevé.

### **2.2.3. Gestion de l'Authentification**

La sécurité de l'accès est assurée par deux extensions Flask clés :
*   **Flask-Login :** Gère le cycle de vie de la session utilisateur (connexion, déconnexion, protection des routes "privées").
*   **Flask-Bcrypt :** Assure le stockage sécurisé des mots de passe en ne sauvegardant que leur "hash" cryptographique, jamais le mot de passe en clair.

## **2.3. Le Pipeline de Données (ETL)**

Le système est alimenté par un pipeline ETL (Extract, Transform, Load) robuste, orchestré par un script Python principal.

### **2.3.1. Orchestration**

Le point d'entrée du processus de données est le script `run_full_pipeline.py`. Il est conçu pour être exécuté de manière autonome, soit manuellement, soit via une tâche planifiée (`cron job`).

### **2.3.2. Extraction (Extract)**

*   **Composition du Portefeuille :** La "source de vérité" pour la liste des actifs et leur mappage est un fichier CSV (`data/tipranks_raw.csv`) lu et parsé avec la librairie **Pandas**.
*   **Prix du Marché :** Les derniers prix de clôture sont récupérés via des appels HTTP à l'API externe **Marketstack**, en utilisant la librairie **requests**.

### **2.3.3. Transformation (Transform)**

Une fois les données extraites, le pipeline exécute plusieurs étapes de nettoyage et de transformation :
*   La fonction `read_and_clean_csv` standardise les noms de colonnes, nettoie les formats de données et parse les informations complexes (comme la colonne "52 Week Range").
*   La logique de conversion de devises est appliquée en se basant sur les instructions fournies dans le fichier CSV (`Marketstack_Currency`) et le taux de change défini dans `config.ini`. C'est à cette étape que la colonne `cad_value` est calculée.

### **2.3.4. Chargement (Load)**

Les données transformées sont insérées dans la base de données MariaDB à l'aide de **SQLAlchemy Core**, qui permet un contrôle fin sur les requêtes SQL :
*   La fonction `synchronize_database` compare le CSV avec la table `titres` et gère la suppression des actifs qui ne sont plus dans le portefeuille (en supprimant d'abord les enregistrements dépendants dans `historique`).
*   La fonction `insert_data` utilise des requêtes `INSERT ... ON DUPLICATE KEY UPDATE` pour insérer les nouveaux relevés de prix ou mettre à jour les informations des titres existants de manière efficace.

## **2.4. L'Infrastructure de Production (VPS)**

L'application est hébergée sur un serveur privé virtuel (VPS) sous **Ubuntu**, avec une architecture moderne basée sur la conteneurisation.

### **2.4.1. Conteneurisation (Docker)**

*   **Service `app` :** Un conteneur Docker, construit sur mesure via un **`Dockerfile`**, qui encapsule l'application Python. L'application est servie par un serveur de production WSGI, **Gunicorn**, optimisé pour gérer plusieurs requêtes simultanées.
*   **Service `db` :** Un conteneur officiel **MariaDB 10.6**, assurant un environnement de base de données stable et isolé.
*   **Orchestration :** L'ensemble des services est défini, configuré et lié par **Docker Compose** (via le fichier `docker-compose.yml`), qui agit comme le chef d'orchestre de l'infrastructure.
*   **Persistance des Données :** Pour garantir qu'aucune donnée de la base de données ne soit perdue lors des mises à jour ou des redémarrages, les fichiers de MariaDB sont stockés dans un **volume Docker nommé**, qui est indépendant du cycle de vie du conteneur.

### **2.4.2. Serveur Web**

**Nginx** est utilisé comme serveur web principal. Il joue le rôle de **reverse proxy** :
*   Il reçoit toutes les requêtes web entrantes sur les ports 80 (HTTP) et 443 (HTTPS).
*   Il transmet les requêtes à l'application tournant dans le conteneur Docker sur le port 8000.
*   Il gère la terminaison SSL/TLS, en servant les certificats gérés par **Certbot** pour assurer une connexion sécurisée (HTTPS).

## **2.5. Le Système de Qualité et de Déploiement (CI/CD)**

L'automatisation est au cœur du projet, gérée par une pipeline de CI/CD hébergée sur **GitHub Actions**.

### **2.5.1. Workflow (`deploy.yml`)**

*   **Déclencheur :** Le workflow est automatiquement lancé à chaque `push` sur la branche `main`.
*   **Job `test` (Intégration Continue) :** Avant tout déploiement, le code est récupéré dans un environnement Linux éphémère. Une suite de tests complète est exécutée avec **Pytest**. Pour garantir une validation réaliste, ce job lance son propre service de base de données **MariaDB** temporaire. Si un seul test échoue, toute la pipeline s'arrête.
*   **Job `deploy` (Déploiement Continu) :** Uniquement si le job `test` réussit, le workflow se connecte au VPS de production via **SSH**. Il exécute ensuite un script qui orchestre le déploiement :
    1.  `git pull` pour récupérer le code validé.
    2.  `docker compose up -d --build` pour reconstruire l'image de l'application avec le nouveau code et redémarrer les services sans interruption majeure.

### **2.5.2. Stratégie de Tests**

*   **Framework :** **Pytest**, pour sa simplicité et sa puissance, ainsi que `pytest-mock` pour simuler les appels externes (API, fichiers).
*   **Environnement de Test Local :** Pour la rapidité, les tests lancés localement utilisent une base de données **SQLite en mémoire**, qui est extrêmement rapide à initialiser et à détruire.
*   **Environnement de Test en CI :** Pour la fiabilité, les tests sur GitHub Actions s'exécutent contre une vraie base de données **MariaDB**, garantissant une compatibilité maximale avec l'environnement de production.
*   **Philosophie TDD :** Le développement des nouvelles fonctionnalités suit, autant que possible, une approche de **Test-Driven Development** : on écrit d'abord un test qui échoue, puis on écrit le code pour le faire passer.

