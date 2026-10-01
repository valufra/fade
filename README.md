<a id="readme-top"></a>

<div align="center">
  <h3 align="center">FADE: Fairness Assessment under Dataset Evolution</h3>

  [![Contributors][contributors-shield]][contributors-url]
  [![Forks][forks-shield]][forks-url]
  [![Stargazers][stars-shield]][stars-url]
  [![Issues][issues-shield]][issues-url]
  [![MIT License][license-shield]][license-url]
</div>

<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
      <ul>
        <li><a href="#built-with">Built With</a></li>
      </ul>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#prerequisites">Prerequisites</a></li>
        <li><a href="#installation">Installation</a></li>
        <li><a href="#configuration">Configuration</a></li>
      </ul>
    </li>
    <li><a href="#framework-overview">Framework Overview</a></li>
    <li><a href="#architecture">Architecture</a></li>
    <li><a href="#project-structure">Project Structure</a></li>
    <li><a href="#workflow">Workflow</a></li>
    <li><a href="#mutation-operators">Mutation Operators</a></li>
    <li><a href="#fairness-symptoms">Fairness Symptoms</a></li>
    <li><a href="#outputs">Outputs</a></li>
    <li><a href="#statistical-analysis">Statistical Analysis</a></li>
    <li><a href="#results-structure">Results Structure</a></li>
    <li><a href="#limitations--notes">Limitations & Notes</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contributing">Contributing</a></li>
    <li><a href="#contact">Contact</a></li>
  </ol>
</details>

<!-- ABOUT THE PROJECT -->

## About The Project

FADE is a framework designed to identify biases in datasets used for training machine learning (ML) models. By using mutation operators, FADE introduces variations in datasets and evaluates the impact of these changes on fairness metrics.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

### Built With

This section lists the main technologies used to develop FADE:

- [![Python][python-shield]][python-url]
- [![Pandas][pandas-shield]][pandas-url]
- [![TensorFlow][tensorflow-shield]][tensorflow-url]
- [![Scikit-learn][scikit-learn-shield]][scikit-learn-url]

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- GETTING STARTED -->

## Getting Started

Here is an example of how you can set up the project locally.
To get a local copy up and running, follow these simple steps.

### Prerequisites

There are no specific prerequisites for using FADE.

- Place your CSV dataset in `data/` (e.g., `data/adult.csv`).
- Ensure the target/outcome column is present (binary or numeric).
- The framework removes rows with missing values and treats `?` as missing.

### Installation

1. Clone the Repository
2. Install Dependencies:
   ```sh
   pip install -r requirements.txt
   ```
3. Run the Framework (interactive):

   ```sh
   cd src
   python main.py
   ```

   - Enter the dataset file name (e.g., `adult.csv`) when prompted.
   - Enter the `target_attribute` (printed list of columns).
   - Select the `positive_label` (default `1` if unsure).

### Configuration

To exclude specific features (e.g., identifiers) from analysis and mutation, you can configure them in `src/config/dataset_config.json`.

Example:

```json
{
  "dataset_name.csv": {
    "ignore_features": ["id", "name", "irrelevant_column"]
  }
}
```

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Framework Overview

FADE applies a set of mutation operators to each non-target feature and evaluates how these changes affect fairness-related symptoms across all potential sensitive attributes. For every mutation, it computes a baseline (pre-mutation) and post-mutation set of symptoms, then exports a streamlined CSV for statistical analysis and the transformed datasets in a hierarchical structure.

- Focuses on dataset-level fairness characteristics (no model training required).
- Works with both numeric and categorical/textual features.
- Provides automated statistical analysis modules to assess significance.

## Architecture

- `src/data/loader.py`: Loads datasets from CSV.
- `src/data/preprocessor.py`: Required preprocessing:
  - Removes rows with missing values (`?` treated as missing).
  - Drops configured ignored features.
  - Applies one-hot encoding when needed.
- `src/analysis/symptom_calculator.py`: Computes 13 dataset-level fairness, distribution, and correlation symptoms across inferred groups.
- `src/mutation/mutation_operator.py`: Implements mutation operators:
  - Numeric: increment/decrement, scaling.
  - Categorical/Textual: category flip, synonym replacement, noise injection.
- `src/utils/result_handlers.py`: Exports streamlined results CSV per dataset.
- `src/utils/file_utils.py`: Creates feature-specific directories and exports transformed datasets.
- `src/utils/analysis_helpers.py`: Maps applicable mutation operators based on feature data types.
- `src/statistical_analysis/run_analysis_rq1.py`: Automated statistical analysis evaluating symptom sensitivity (Wilcoxon Signed-Rank Test + rank-biserial effect size with FDR Benjamini-Hochberg adjustment) and Composite Fragility Score (CFS) ranking.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Project Structure

Top-level directories:

- `src/`: Core framework code for dataset-level fairness symptom analysis, mutation application, and statistical testing.
  - `analysis/`: Symptom calculation modules (`symptom_calculator.py`).
  - `config/`: Configuration files (e.g., `dataset_config.json` for feature exclusion).
  - `data/`: Dataset loading and preprocessing pipelines (`loader.py`, `preprocessor.py`).
  - `mutation/`: Feature mutation operators (`mutation_operator.py`).
  - `statistical_analysis/`: Automated sensitivity and fragility analysis (`run_analysis_rq1.py`).
  - `utils/`: Result export, directory structuring, and operator helpers.
- `data/`: Benchmark datasets used by the main framework (`adult.csv`, `bank.csv`, `compas.csv`, `diabetic.csv`).
- `results/`: Outputs produced by `src/main.py` and the statistical analysis pipeline:
  - Streamlined baseline results per dataset (`results_adult.csv`, `results_bank.csv`, `results_compas.csv`, `results_diabetic.csv`) and aggregated `results.csv`.
  - Fragility rankings (`fade_analysis.csv`) and symptom sensitivity statistics (`symptom_sensitivity_results.csv`).
  - Hierarchical feature mutation folders per dataset (`adult/`, `bank/`, `compas/`, `diabetic/`) containing transformed CSV datasets.
  - `dataset-RQ2/`: Transformed datasets curated for downstream model fairness evaluation across all benchmark datasets.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Workflow

1. Load dataset and remove missing values (`clean_data_only`).
2. Prompt user for `target_attribute` and `positive_label`.
3. Compute baseline symptoms for all features except the target.
4. For each feature:
   - Determine applicable mutation operators (based on data type).
   - Apply mutation to the clean dataset.
   - Compute post-mutation symptoms for all potential sensitive attributes using the mutated data.
   - Record per-symptom differences and print a change summary.
5. Export:
   - Streamlined results CSV for statistical analysis.
   - Transformed datasets in hierarchical feature-specific directories.

Notes:

- Change reporting threshold: prints changes when `|Δ| > 0.01`, flags “SIGNIFICANT CHANGE” at `|Δ| > 0.1` (console only; statistical tests use p-values).

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Mutation Operators

Implemented in `src/mutation/mutation_operator.py`:

- `increment_decrement_feature` (numeric): Random ±5% of feature range on ~20% rows.
- `scale_values` (numeric): Multiply by factor in `[0.8, 1.2]` on ~20% rows, preserves integer dtype.
- `category_flip` (categorical): Reassign ~15% of instances in each category to another category.
- `replace_synonyms` (textual): In ~10% rows, replace ~15% of words with WordNet synonyms.
- `add_noise` (textual/categorical strings): Inject simple character-level noise into ~10% rows.
  - For categorical strings, applies consistent mutations per unique category (<=45 unique values).

Applicable operators are selected per column via `src/utils/analysis_helpers.py`.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Fairness Symptoms

Calculated in `src/analysis/symptom_calculator.py`:

- **Distribution/Diversity**:
  - `Gini Index`: Gini coefficient of inequality for category frequency distributions.
  - `Shannon Entropy`: Information entropy of category frequencies.
  - `Simpson Diversity`: Simpson index measuring the probability that two random instances share the same category.
  - `Imbalance Ratio`: Ratio of minimum category count to maximum category count.
  - `Kurtosis`: Measure of distribution tailedness (numeric only).
  - `Skewness`: Measure of distribution asymmetry (numeric only).
- **Relationship to Target**:
  - `Mutual Information`: Dependency between feature and target outcome (via scikit-learn).
  - `Kendall Tau`: Rank correlation between numeric feature and target (numeric only).
- **Ground-Truth Fairness Metrics (Group-based)**:
  - `Data Statistical Parity`: Absolute difference in positive outcome probability between groups $|P(Y=1|\text{Unpriv}) - P(Y=1|\text{Priv})|$.
  - `Unprivileged Positive Probability`: Positive outcome probability in the unprivileged group $P(Y=1|\text{Unpriv})$.
  - `Privileged Positive Probability`: Positive outcome probability in the privileged group $P(Y=1|\text{Priv})$.
  - `Unprivileged Group Unbalance`: Ratio of unprivileged positive rate to the overall positive rate.
  - `Privileged Group Unbalance`: Ratio of privileged positive rate to the overall positive rate.

Group inference: Continuous features (>20 unique values) are split at the median ($\le \text{median}$ vs. $> \text{median}$); categorical/discrete features identify categories with the minimum and maximum positive label rates as unprivileged and privileged respectively.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Outputs

1. **Streamlined Results CSV (per symptom)**:
   - Path: `results/results_{datasetName}_{timestamp}.csv` (and baseline `results/results_{datasetName}.csv`)
   - Columns:
     - `column`, `dataset`, `column_type`, `mutation_type`, `is_sensitive`, `sensitive_attr_analyzed`
     - `symptom_name`, `pre_symptom_value`, `post_symptom_value`
     - `symptom_difference`, `symptom_abs_difference`, `symptom_percent_change`
2. **Transformed Datasets (per mutated feature and operator)**:
   - Path pattern: `results/{dataset}/feature_{featureName}_{timestamp}/transformed_{dataset}_{column}_{mutation}.csv`
3. **Statistical Analysis Outputs**:
   - `results/results.csv`: Aggregated streamlined results across all datasets.
   - `results/fade_analysis.csv`: Composite Fragility Score (CFS) per feature and mutation, identifying high-risk fairness-fragile features.
   - `results/symptom_sensitivity_results.csv`: Sensitivity statistics per symptom (Wilcoxon test p-values, FDR-adjusted `p_adj`, rank-biserial effect sizes, and trend directions).

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Statistical Analysis

Statistical analysis is automated via `src/statistical_analysis/run_analysis_rq1.py`, operating on the streamlined results CSVs.

### RQ1 Analysis (FADE)

```bash
python src/statistical_analysis/run_analysis_rq1.py
```

The script automatically executes a two-stage analysis:

1. **Dynamic Aggregation**: Automatically aggregates all `results_*.csv` files in `results/` into a unified `results/results.csv`.
2. **Part A — Symptom Sensitivity Analysis**:
   - Evaluates which fairness metrics are most reactive under dataset mutations.
   - Applies the non-parametric Wilcoxon Signed-Rank test comparing pre- and post-mutation symptom values across all mutation events.
   - Computes rank-biserial correlation ($r$) as effect size.
   - Adjusts p-values using the Benjamini-Hochberg False Discovery Rate (FDR) procedure ($\alpha = 0.05$) to control for multiple testing.
   - Output: `results/symptom_sensitivity_results.csv`.
3. **Part B — Composite Fragility Score (CFS)**:
   - Quantifies the degree of fairness fragility for each dataset feature under mutation.
   - Calculates the Composite Fragility Score based on absolute percentage changes across symptoms (`CFS_Mean`, `CFS_Median`, `Max_Symptom_Change`).
   - Identifies high-risk "fairness-fragile" features (top 20% quantile of reactivity for each dataset).
   - Output: `results/fade_analysis.csv`.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Results Structure

Outputs are organized under `results/`:

- `results/<dataset>/feature_<feature>/transformed_<dataset>_<column>_<mutation>.csv` (hierarchical transformed datasets)
- `results/dataset-RQ2/<dataset>/transformed_<dataset>_<column>_<mutation>.csv` (transformed sets for downstream model experiments)
- `results/results_<dataset>.csv` (per-dataset streamlined symptom results)
- `results/results.csv` (aggregated multi-dataset results)
- `results/fade_analysis.csv` (feature fragility rankings & CFS scores)
- `results/symptom_sensitivity_results.csv` (symptom sensitivity & Wilcoxon statistical results)

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Limitations & Notes

- Preprocessing strictly removes rows with missing values and treats `?` as missing.
- Group inference may fail on degenerate splits (e.g., empty halves); such metrics return `None`.
- Console “SIGNIFICANT CHANGE” uses a heuristic (`|Δ| > 0.1`); statistical significance relies on tests (p-values).
- Symptom calculations are dataset-level (no model predictions required).
- Mutation parameters are configurable but have sensible defaults aligned with dataset realism.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- LICENSE -->

## License

Distributed under the MIT License. See `LICENSE` for more information.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- CONTRIBUTING -->

## Contributing

Contributions are what makes the open source community such an amazing place to learn, inspire and create. Any contributions you make will be **very much appreciated**.

If you have a suggestion that could improve this project, please fork the repository and create a pull request. You can also simply open an issue with the ‘enhancement’ tag.
Don't forget to give the project a star! Thanks again!

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m ‘Add some AmazingFeature’`)
4. Push on the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- CONTACT -->

## Contact

Alfonso Cannavale - [LinkedIn](https://www.linkedin.com/in/alfonso-cannavale-62150b229/)

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->

[contributors-shield]: https://img.shields.io/github/contributors/alfcan/FADE.svg?style=for-the-badge
[contributors-url]: https://github.com/alfcan/FADE/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/alfcan/FADE.svg?style=for-the-badge
[forks-url]: https://github.com/alfcan/FADE/network/members
[stars-shield]: https://img.shields.io/github/stars/alfcan/FADE.svg?style=for-the-badge
[stars-url]: https://github.com/alfcan/FADE/stargazers
[issues-shield]: https://img.shields.io/github/issues/alfcan/FADE.svg?style=for-the-badge
[issues-url]: https://github.com/alfcan/FADE/issues
[license-shield]: https://img.shields.io/github/license/alfcan/FADE.svg?style=for-the-badge
[license-url]: https://github.com/alfcan/FADE/blob/master/LICENSE
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=for-the-badge&logo=linkedin&colorB=555
[linkedin-url]: https://www.linkedin.com/in/alfonso-cannavale-62150b229/
[python-shield]: https://img.shields.io/badge/python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54
[python-url]: https://www.python.org/
[pandas-shield]: https://img.shields.io/badge/pandas-150458?style=for-the-badge&logo=pandas&logoColor=white
[pandas-url]: https://pandas.pydata.org/
[tensorflow-shield]: https://img.shields.io/badge/tensorflow-FF6F00?style=for-the-badge&logo=tensorflow&logoColor=white
[tensorflow-url]: https://www.tensorflow.org/
[scikit-learn-shield]: https://img.shields.io/badge/scikit--learn-F7931E?style=for-the-badge&logo=scikit-learn&logoColor=white
[scikit-learn-url]: https://scikit-learn.org/
