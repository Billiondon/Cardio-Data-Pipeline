#  Cardio Data Pipeline (`dbt` + `BigQuery`)
---

## Problématique

En recherche clinique , la valeur d'un travail scientifique dépend la qualité des données qui l'alimentent.
Souvent le chercheur est confronté à des encodages numériques, des formats hétérogènes, valeurs aberrantes biaisant la lecture clinique ou entrainement de modele.

Ce dépôt répond à un double objectif :
* **Sur le plan médical :** traduire des variables brutes (pression artérielle, fréquence cardiaque, statut coronarien) en indicateurs cliniques lisibles et encadrés par des règles de **plausibilité physiologique**, afin d’exclure toute valeur biologiquement incohérente avant l’analyse.
* **Sur le plan data & pédagogique :** concevoir un pipeline modulaire  qui montre concrètement comment passer d’un fichier CSV brut à une donnée prête pour le Machine Learning sans effet « boîte noire ».

---

## Méthode & Architecture

Le pipeline suit une progression logique en trois étapes, où chaque transformation est isolée, documentée et vérifiée automatiquement :

```text
[ seeds/heart_disease.csv ] ──▶ [ staging: stg_heart_disease ] ──▶ [ Tests qualité & physiologie ]
        (Donnée brute)               (Médicalisation & nettoyage)          (Contrôle avant ML)

## Auteur
El Bachir ROCHD, interne en médecine  
