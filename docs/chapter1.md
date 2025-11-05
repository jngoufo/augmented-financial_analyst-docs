# **Chapitre 1 : Vision et Objectifs du Projet**

## **1.1. Introduction : La Problématique**

La gestion moderne d'un portefeuille d'investissement, bien qu'accessible, présente une complexité croissante. Les investisseurs actifs interagissent avec de multiples plateformes : courtiers pour les transactions, services d'analyse pour la recherche (ex: TipRanks), et sources de données de marché pour le suivi des prix (ex: Marketstack).

Cette fragmentation crée une charge de travail manuelle significative et rend difficile l'obtention d'une vue d'ensemble unifiée, précise et historique de la performance du portefeuille. Répondre à des questions simples comme "Quelle a été la performance réelle de mon titre X, toutes devises confondues ?" ou "Quels sont les actifs qui surperforment aujourd'hui ?" nécessite une consolidation fastidieuse et sujette aux erreurs.

## **1.2. La Solution : Le Projet "Augmented Analyst"**

Le projet "Augmented Analyst" a été conçu pour résoudre cette problématique en fournissant une plateforme d'analyse de portefeuille privée, centralisée et automatisée.

L'application agit comme un hub personnel qui :
1.  **Consolide** la composition du portefeuille à partir d'une source de données unique et contrôlée.
2.  **Enrichit** automatiquement ces données avec les prix du marché les plus récents.
3.  **Analyse** l'évolution et la performance du portefeuille, tant au niveau global que pour chaque titre individuel.
4.  **Présente** ces informations de manière claire et visuelle via une interface web privée et sécurisée.

L'objectif final est de transformer des données brutes et dispersées en informations exploitables, permettant à l'utilisateur de prendre des décisions d'investissement plus éclairées et de suivre la performance de ses actifs avec une confiance absolue.

## **1.3. Objectifs Clés**

Les objectifs du projet se divisent en deux catégories : les objectifs fonctionnels (ce que l'application fait) et les objectifs non-fonctionnels (les principes de qualité qui régissent son fonctionnement).

### **1.3.1. Objectifs Fonctionnels**

*   **Importation du Portefeuille :** L'application doit pouvoir importer et synchroniser la composition complète du portefeuille (liste des titres, quantités) à partir d'un fichier de configuration au format CSV.
*   **Enrichissement Automatisé des Données :** Le système doit, de manière autonome et quotidienne, récupérer les derniers prix de clôture pour chaque titre via une API financière externe (Marketstack).
*   **Historisation des Données :** Chaque relevé de prix quotidien doit être stocké de manière persistante, créant un historique complet qui permet de suivre l'évolution de la valeur de chaque titre et du portefeuille global dans le temps.
*   **Accès Sécurisé :** L'accès à toutes les données et analyses doit être protégé et strictement limité à un unique utilisateur authentifié via un système de connexion.
*   **Présentation Visuelle :**
    *   Un **Dashboard Global** doit présenter une vue d'ensemble du portefeuille, incluant sa valeur totale, sa performance journalière, et des graphiques d'évolution.
    *   Des **Pages de Détail** doivent permettre une analyse approfondie pour chaque titre, incluant un graphique de performance historique et des données techniques.
*   **Analyse de Performance :**
    *   Calculer la performance journalière du portefeuille global.
    *   Identifier et classer les titres les plus et les moins performants du jour ("Top/Flop Performers").
    *   Analyser la position du prix actuel de chaque titre par rapport à ses plus hauts et plus bas sur 52 semaines.

### **1.3.2. Objectifs Non-Fonctionnels**

*   **Fiabilité et Précision :** Les données financières affichées doivent être irréprochables. Les calculs doivent être exacts, et les incohérences de données (ex: devises, tickers) doivent être gérées de manière robuste. C'est l'objectif de qualité le plus important.
*   **Automatisation :** Le système doit fonctionner sans intervention manuelle quotidienne. Les mises à jour de données et les déploiements de code doivent être entièrement automatisés.
*   **Maintenabilité :** L'architecture logicielle doit être propre, modulaire et facile à comprendre pour permettre des évolutions futures sans avoir à tout reconstruire.
*   **Sécurité :** L'application doit suivre les bonnes pratiques de sécurité web pour protéger les données de l'utilisateur et les clés d'accès aux services externes.

## **1.4. Principes Directeurs et Philosophie d'Architecture**

Pour atteindre ces objectifs, le projet "Augmented Analyst" a été construit sur une fondation de principes DevOps modernes. Ces décisions d'architecture ne sont pas accidentelles ; elles sont le fruit d'une volonté de construire un système de qualité professionnelle.

### **1.4.1. L'Infrastructure comme du Code (IaC) : La Fiabilité par la Conteneurisation**
Le projet rejette l'approche fragile de la configuration manuelle d'un serveur. L'intégralité de l'application (le code Python, la base de données) est encapsulée dans des conteneurs **Docker**.
*   **Pourquoi ?** Cette approche garantit que l'environnement de développement, de test et de production est **strictement identique**, éliminant la cause la plus fréquente de bugs : "ça marchait sur ma machine". La configuration est définie dans des fichiers (`Dockerfile`, `docker-compose.yml`), ce qui la rend versionnable, reproductible et fiable.

### **1.4.2. L'Automatisation Totale (CI/CD) : Le Robot Développeur**
Le projet n'autorise aucun déploiement manuel. Chaque modification du code est gérée par une pipeline de **CI/CD (Intégration Continue / Déploiement Continu)** orchestrée par **GitHub Actions**.
*   **Pourquoi ?** L'automatisation du déploiement via un "robot" élimine le risque d'erreur humaine, garantit que chaque déploiement suit exactement la même procédure, et permet de livrer des améliorations de manière rapide et sereine.

### **1.4.3. Le Filet de Sécurité Permanent (Tests Automatisés)**
Aucune ligne de code n'est "poussée" en production sans avoir été validée par une suite de tests automatisés rigoureux, écrits avec **Pytest**.
*   **Pourquoi ?** Les tests sont le "garde du corps" de l'application. Ils protègent contre les régressions, valident que les nouvelles fonctionnalités se comportent comme prévu, et servent de documentation vivante du fonctionnement du système. C'est la garantie de la maintenabilité à long terme.

### **1.4.4. La Source de Vérité Explicite : L'Humain au Contrôle**
Pour les logiques métier complexes et sujettes à l'interprétation (comme la correspondance des tickers entre différentes sources ou la gestion des devises), le projet adopte un principe clair : le code ne doit pas "deviner".
*   **Pourquoi ?** La fiabilité des données prime sur la "magie" du code. C'est l'utilisateur, via des fichiers de configuration enrichis (le CSV de portefeuille avec ses colonnes de mappage, le fichier `config.ini`), qui fournit des **instructions explicites** au système. Le code devient alors un exécutant simple et fiable de ces instructions, ce qui rend le comportement du pipeline prévisible et facile à déboguer.
