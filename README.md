# Cardio Data Pipeline (`dbt` + `BigQuery`)

---

## Problématique

En recherche clinique, la valeur d'un travail scientifique dépend de la qualité des données qui l'alimentent.
Le chercheur est souvent confronté à des encodages numériques, des formats hétérogènes et des valeurs aberrantes qui biaisent la lecture clinique ou l'entraînement de modèles.

Ce dépôt est un exercice qui poursuit un double objectif :

* **Sur le plan médical :** traduire des variables brutes (TA, FC maximale à l'effort, statut coronarien) en variables cliniques lisibles
* **Sur le plan data et pédagogique :** concevoir un pipeline modulaire qui montre comment passer d'un fichier CSV brut à une donnée exploitable, sans effet black box.

> NB : données publiques uniquement, aucune donnée patient réelle.

---

## Méthode 

À partir d'une archive de référence de données cliniques brutes (UCI Machine Learning Repository, hébergé par l'Université de Californie à Irvine), ce pipeline traite la base Cleveland de 303 patients, 14 variables retenues parmi les 76 attributs d'origine.

Cette base présente cependant plusieurs obstacles :

* Abréviations sans unité (`trestbps`, `thalach`)
* Codes sans valeur explicite (sexe codé 1/0, atteinte coronarienne codée de 0 à 4)
* Valeurs manquantes (`?` dans le fichier d'origine)
* Risque de valeurs aberrantes

Une étape de nettoyage comble l'écart entre un fichier d'archive brut et une donnée médicalement intelligible, sans imputer ni inventer de valeur :

1. **Renommer avec l'unité de mesure** : `trestbps` → `resting_bp_mmhg`, `chol` → `cholesterol_mg_dl`, `thalach` → `max_heart_rate_bpm` (FC maximale à l'effort).
2. **Décoder le sexe** : 1/0 → M/F.
3. **Binariser la cible** : `has_heart_disease = (num > 0)`. Les valeurs 1 à 4 sont regroupées pour obtenir une réponse binaire (0 = absence, 1 = présence), convention classique sur ce jeu.
4. **Typer en entier** `ca` : on respecte sa nature discrète malgré les cases vides.
5. **Garder les NULL** de `ca` (4 valeurs manquantes) : on laisse les valeurs vides plutôt que d'inventer une valeur à la place du clinicien.
6. **Ajouter un `id` de ligne** : repère unique par ligne (Cleveland ne fournit pas d'identifiant patient).
7. **Ajouter des tests automatiques** : identifiant, valeurs de sexe, et fourchettes larges de pression artérielle (50-300 mmHg) et de fréquence cardiaque (40-250 bpm) pour repérer les anomalies grossières

Donc j'ai commencé par stocker une table source brut en données Cloud BigQuery via dbt seed (seeds/heart_disease.csv).
Cela permet de garder une base de référence reproductible.
Ensuite, j’ai créé un modèle SQL dans dbt (models/cardio/stg_heart_disease.sql) qui s’exécute directement dans BigQuery pour nettoyer et transformer cette table brute en une table propre 
Enfin, j’ai ajouté des tests automatiques dans dbt (schema.yml et tests/bp_hr_plausible.sql) exécutés sur BigQuery à savoir:
-Vérification que l’id est unique et non nul
-Vérification que le sexe vaut bien M ou F
-Vérification que la pression artérielle (50–300 mmHg)
-Vérification quela fréquence cardiaque (40–250 bpm) 

.

**Périmètre** : pour ce premier exercice, **7 variables sur 14** sont conservées (`age`, `sex`, `trestbps`, `chol`, `thalach`, `ca`, `num`), plus l'identifiant. Les 7 autres (`cp`, `fbs`, `restecg`, `exang`, `oldpeak`, `slope`, `thal`) sont écartées par choix pédagogique, pas pour une raison clinique.

| Élément | Rôle |
|---|---|
| `seeds/heart_disease.csv` | Données brutes UCI (Cleveland, 303 patients) |
| `models/cardio/stg_heart_disease.sql` | Renommage explicite des colonnes, sexe encodé M/F, cible binaire `has_heart_disease` |
| `models/cardio/schema.yml` | Tests `unique`, `not_null`, `accepted_values` |
| `tests/bp_hr_plausible.sql` | Plausibilité physiologique (PA 50-300 mmHg, FC 40-250 bpm) |

Chaque transformation est isolée, documentée et vérifiée automatiquement :

```text
[ seeds/heart_disease.csv ] ──▶ [ staging: stg_heart_disease ] ──▶ [ Tests qualité et physiologie ]
        (Donnée brute)              (Nettoyage et encodage lisible)          (Contrôle avant analyse / ML)
```
## Aperçu : avant / après

**Avant** : `seeds/heart_disease.csv` (abréviations, sexe en 1/0, cible de 0 à 4, valeur manquante vide)

| id  | age | sex | trestbps | chol | thalach | ca      | num |
|-----|-----|-----|----------|------|---------|---------|-----|
| 1   | 63  | 1   | 145      | 233  | 150     | 0       | 0   |
| 2   | 67  | 1   | 160      | 286  | 108     | 3       | 2   |
| 3   | 67  | 1   | 120      | 229  | 129     | 2       | 1   |
| 4   | 37  | 1   | 130      | 250  | 187     | 0       | 0   |
| 288 | 58  | 1   | 125      | 220  | 144     | (vide)  | 0   |

**Après** : `stg_heart_disease` (noms explicites avec unités, sexe décodé, cible binaire, NULL conservé)

| patient_id | age  | sex | resting_bp_mmhg | cholesterol_mg_dl | max_heart_rate_bpm | nb_vessels_colored | has_heart_disease |
|------------|------|-----|-----------------|-------------------|--------------------|--------------------|-------------------|
| 1          | 63.0 | M   | 145.0           | 233.0             | 150.0              | 0                  | false             |
| 2          | 67.0 | M   | 160.0           | 286.0             | 108.0              | 3                  | true              |
| 3          | 67.0 | M   | 120.0           | 229.0             | 129.0              | 2                  | true              |
| 4          | 37.0 | M   | 130.0           | 250.0             | 187.0              | 0                  | false             |
| 288        | 58.0 | M   | 125.0           | 220.0             | 144.0              | NULL               | false             |

---

## Stack

* **dbt (Fusion)** : transformations SQL modulaires et tests automatisés de qualité et de plausibilité clinique
* **Google BigQuery** : entrepôt de données cloud pour les tables brutes et préparées
* **GitHub (Pull Requests)** : versioning et traçabilité des règles métier et des seuils cliniques

---

## Lancer le projet

```bash
dbt seed --select heart_disease
dbt build --select stg_heart_disease
```

---

## Limites

* Validité externe limitée : jeu de données ancien (UCI Heart Disease, 1989) et de petite taille (303 patients).
* Les seuils de plausibilité sont volontairement larges : ils détectent les erreurs de saisie, pas les valeurs pathologiques. Ils sont propres à ce projet et ne viennent pas de la documentation UCI.
* Sous-ensemble de 7 variables sur 14 : le modèle n'est pas encore utilisable tel quel pour du machine learning.

---

## Source 

Janosi, A., Steinbrunn, W., Pfisterer, M., Detrano, R. (1989). *Heart Disease*. UCI Machine Learning Repository. Licence CC BY 4.0.

## Auteur

El Bachir ROCHD, interne en médecine | ML appliqué 
