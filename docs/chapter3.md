# **Chapitre 3 : Guide d'Utilisation et de Maintenance**

Ce chapitre est un guide opérationnel. Il décrit les procédures standard pour développer, maintenir et administrer l'application "Augmented Analyst".

## **3.1. Le Workflow de Développement Idéal**

Toute modification du projet, qu'il s'agisse d'une correction de bug, de l'ajout d'une nouvelle fonctionnalité ou d'une simple mise à jour de texte, doit suivre ce processus rigoureux pour garantir la stabilité de la production.

#### **3.1.1. Création d'une Branche de Travail**
Ne travaillez jamais directement sur la branche `main`. Chaque nouvelle tâche commence par la création d'une branche dédiée depuis la version la plus à jour de `main`.

```bash
# S'assurer que la branche `main` locale est à jour
git checkout main
git pull origin main

# Créer et basculer sur une nouvelle branche descriptive
git checkout -b <type_de_changement>/<description-courte>
```
*Exemples de noms de branche :* `feature/add-new-chart`, `fix/login-page-bug`.

#### **3.1.2. Modification du Code Local**
Effectuez toutes les modifications de code nécessaires dans votre environnement de développement local (ex: VS Code).

#### **3.1.3. Validation par les Tests Locaux**
Avant de soumettre votre travail, exécutez l'intégralité de la suite de tests pour vous assurer que vos changements n'ont introduit aucune régression.  Cette commande doit être exécutée depuis la racine du dépôt de l'application (qa-automated-pipeline).

```bash
# Depuis la racine du projet applicatif (qa-automated-pipeline),
# avec son environnement virtuel (venv) activé
pytest -v
```
Tous les tests doivent passer (ou être `skipped` de manière intentionnelle).

#### **3.1.4. Sauvegarde du Travail (`Commit`)**
Sauvegardez vos changements dans l'historique Git avec un message de commit clair et concis, respectant la convention "Conventional Commits".

```bash
# Ajouter les fichiers modifiés
git add .

# Créer le commit
git commit -m "type(scope): description du changement"
```
*Exemples :* `feat(dashboard): Add 52-week analysis panel`, `fix(pipeline): Correct currency conversion logic`.

#### **3.1.5. Partage et Revue (Pull Request)**
Poussez votre branche de travail sur le dépôt GitHub et ouvrez une "Pull Request" (PR).

```bash
git push --set-upstream origin <nom_de_votre_branche>
```
Sur GitHub, créez la Pull Request en ciblant la branche `main`. La PR est l'opportunité de décrire vos changements et de laisser la CI/CD exécuter les tests une dernière fois dans un environnement propre.

#### **3.1.6. Déploiement Automatisé**
Une fois la Pull Request validée (revue et passage des tests en CI), fusionnez-la dans `main`. Cette action déclenchera automatiquement le workflow de déploiement, qui mettra à jour l'application en production sans aucune autre intervention manuelle.

## **3.2. Procédures de Maintenance des Données**

#### **3.2.1. Mettre à Jour la Composition du Portefeuille**
*   **Quand :** Lorsque vous achetez de nouvelles actions, en vendez, ou modifiez des quantités.
*   **Procédure :**
    1.  Ouvrez le fichier `data/tipranks_raw.csv` sur votre **machine locale**.
    2.  Modifiez, ajoutez ou supprimez les lignes nécessaires.
    3.  **Crucial :** Pour toute nouvelle ligne, assurez-vous de remplir les colonnes de mappage `Marketstack_Ticker` et `Marketstack_Currency` après avoir fait la recherche sur le site de Marketstack.
    4.  Suivez le **Workflow de Développement Idéal** (commit, push, merge). Le pipeline de déploiement détectera le changement sur le CSV et lancera automatiquement le script d'importation après le déploiement du code.

#### **3.2.2. Mettre à Jour Manuellement les Prix du Marché**
*   **Quand :** Pour un besoin ponctuel de rafraîchissement des données en dehors de l'exécution planifiée.
*   **Procédure :**
    1.  Connectez-vous en SSH au VPS.
    2.  Naviguez vers le répertoire du projet : `cd /var/www/qa-automated-pipeline`.
    3.  Exécutez la commande : `docker compose exec app python code_source_simule/import_data.py`.

## **3.3. Administration du Serveur de Production (VPS)**

#### **3.3.1. Vérifier l'État de l'Application**
*   **Voir les conteneurs actifs :** `docker ps` (Doit montrer `qa-automated-pipeline-app-1` et `qa-automated-pipeline-db-1` avec le statut `Up`).
*   **Logs de l'application (Gunicorn/Flask) :** `cd /var/www/qa-automated-pipeline && docker compose logs -f app` (`-f` pour suivre en direct).
*   **Logs d'erreurs du serveur web :** `sudo tail -f /var/log/nginx/error.log`.

#### **3.3.2. Redémarrer les Services**
*   **Redémarrage complet (App + BDD) :** `cd /var/www/qa-automated-pipeline && docker compose restart`.
*   **Redémarrage de l'application seule :** `cd /var/www/qa-automated-pipeline && docker compose restart app`.
*   **Redémarrage de Nginx :** `sudo systemctl restart nginx`.

#### **3.3.3. Gérer le Pipeline Automatisé**
*   **Afficher les tâches planifiées :** `crontab -l`.
*   **Modifier les tâches planifiées :** `crontab -e`.
*   **Vérifier les logs de la dernière exécution :** `cat /var/log/cron-pipeline.log`.

#### **3.3.4. Interagir avec la Base de Données**
1.  Connectez-vous en SSH au VPS.
2.  Naviguez vers le répertoire du projet : `cd /var/www/qa-automated-pipeline`.
3.  Chargez les variables d'environnement : `source .env`.
4.  Lancez le client MariaDB : `docker compose exec db mysql -u"$DB_PROD_USER" -p"$DB_PROD_PASSWORD" "$DB_PROD_NAME"`.

## **3.4. Gestion des Dépendances**

Pour ajouter une nouvelle librairie Python (ex: `nouvelle-librairie`):
1.  **Sur votre machine locale**, avec le `venv` activé, installez la librairie : `pip install nouvelle-librairie`.
2.  Mettez à jour le fichier `requirements.txt` pour qu'il contienne la nouvelle dépendance et sa version. C'est la commande la plus importante pour garantir la reproductibilité.
    ```bash
    pip freeze > requirements.txt
    ```
3.  Vérifiez que l'application fonctionne toujours en lançant les tests locaux (`pytest -v`).
4.  Suivez le **Workflow de Développement Idéal** (commit, push, etc.). Le pipeline CI/CD reconstruira l'image Docker avec la nouvelle librairie (`--build`), rendant la dépendance disponible en production.

