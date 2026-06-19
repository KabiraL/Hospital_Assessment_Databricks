# Which U.S. hospitals deliver better outcomes at lower cost, and why CMS star ratings miss 2 out of 5 of them.



>Stack: SQL | Python (pandas, scikit-learn, statsmodels) | Databricks | Tableau  
>Methods: outcome scoring | OLS & random forests | hypothesis testing | CMS star benchmark  
>Scope: 7 public CMS datasets | 2,306-hospital dataset  



#### CMS star ratings don't tell the whole story. The best-value providers are efficient exceptions that star ratings may miss, and only the cost half of value can be reliably predicted.
#### A Medicare hospital value analysis in SQL, Python, Databricks, and Tableau, drawing on ~5,000 hospitals from 7 public CMS datasets, analyzing the 2,306 hospitals that had star ratings, outcomes data and cost data. The objective was to find which hospitals deliver higher outcomes at lower cost. The analysis also shows what, if anything, sets them apart. Mortality, readmission, safety, spending, effectiveness, and patient experience data were analyzed to identify high-value providers.

<hr style="border:2px solid gray">

### The bottom line
#### Question: Which U.S. hospitals deliver higher outcomes at lower costs, and what distinguishes them?
#### Answer: Clinical outcomes and cost are only weakly linked. You may or may not see better outcomes when costs increase, so the genuinely high-value hospitals (above-median outcomes at below-median Medicare per-episode spending) are off-trend exceptions. They can't be predicted from operational, patient-experience, or structural data because that data captures how a hospital is run more than how its patients fare. What **can** be moderately predicted is cost on its own. A model yields ~84% precision on held-out data. The details are in the technical section below. The official CMS star ratings, which never factor in cost, miss roughly two in five high-value hospitals.

<div align="center">
<img width="900" alt="Outcome/Cost Scatterplot" src="https://github.com/KabiraL/Hospital_Assessment_Databricks/blob/33ea1d545e0c447129e1b62e0fe1ca48bc994701/Data/outcome_cost_scatter.png" />
</div>

Each dot is a hospital, plotted by clinical outcome and cost. The high-value group (higher outcomes at lower cost in the lower right) is the one we are concerned about.

### What I found

- In healthcare, higher cost doesn't necessarily mean better outcomes.  The two are only weakly, and slightly positively, linked. Genuinely high-value hospitals (higher outcomes with lower cost) are exceptions that run against the grain.
- I found that cost is predictable. Clinical outcomes are not predictable from this data, however. Factors such as patient experience, ED efficiency, hospital size, and location explain cost moderately, but only weakly explain clinical outcomes. Two independent methods (linear regression and random forests) agree, which points to a real limitation of the data rather than issues with the statistical models. This makes sense. Patient experience, ED efficiency, size, and location measure how a hospital is run. And how a hospital is run is a weak predictor of how its patients fare.
- With this data, a cost screen works, but a value screen can't. The tuned model identifies likely low-cost hospitals with ~84% precision on held-out data, making it a workable cost screen, even though the high-value combination itself resists prediction.  

My outcome score is built from the same clinical measures the star rating relies on, so stars and value largely agree on quality. The differences are almost entirely about cost, which is the single dimension from my analysis that stars structurally can’t see.
There are 216 high-value hospitals (about 2 in 5 of all high-value hospitals) which are rated only 1–3 stars because the star system never weighs cost. Conversely, 61% of top-rated (4–5 star) hospitals don't meet the high value standard once cost is counted.

<div align="center">
<img width="900" alt="Value/Star Matrix" src="https://github.com/KabiraL/Hospital_Assessment_Databricks/blob/9e9ec77f594bcc481319414a84e8faca621f35a5/Data/value_star_matrix.png" />
</div>

#### "Hidden value" (high-value hospitals the stars underrate) and "Reputation premium" (top-rated hospitals that aren't high value) are exactly the cells star ratings alone can't reveal.

#### Where the hidden high-value hospitals are
<div align="center">
<img width="900" alt="Hidden Value Map" src="https://github.com/KabiraL/Hospital_Assessment_Databricks/blob/9e9ec77f594bcc481319414a84e8faca621f35a5/Data/hidden_value_map_green.png" />
</div>

High-value hidden hospitals, mapped nationwide.

**[See more about high-value hospitals on Tableau Public →](https://public.tableau.com/app/profile/leah.kabira/viz/HiddenValueHospitals/HiddenValueandStandouts)**

Value tilts modestly by region and ownership, but variation within every group is large, so the map shows where to look, not a guarantee.

### Why it matters
Healthcare affordability is a top domestic concern for Americans, and "value" (outcomes relative to cost) is the metric payers, health systems, and care-coordination teams increasingly manage to. This analysis reframes that work. True high-value hospitals can't be inferred from price, patient reviews, or reputation, because none of those signals reliably tracks clinical quality, only clinical efficiency. Identifying high-value hospitals requires measuring outcomes directly. But the cost half of the equation can be screened quickly and reliably, and the hospitals that pair strong outcomes with low cost are precisely the efficient providers a value-conscious organization would overlook if it screened on star ratings alone. This analysis uses Medicare spending per episode (the total spend underlying CMS's MSPB measure) as the cost lens because it is a fast, risk-adjusted proxy, though it is a narrower measure than full total cost of care.

Readers can look up any hospital on Medicare's official consumer tool, [Care Compare](https://www.medicare.gov/care-compare/). This provides a comprehensive list of hospitals with their overall star ratings and patient survey results.

**[View the executive presentation →](https://github.com/KabiraL/Hospital_Assessment_Databricks/blob/998cbe280ad9c72a8c3e718c2f4c79bbf5c7db75/Hospital_Value_Executive_Deck.pdf)**

Full methodology and model diagnostics are in the notebooks below.

### How I built it
The analysis runs in two notebooks:

1. Hospital Data SQL Analysis.ipynb contains data cleaning, the clinical-outcome scoring system, and exploratory analysis. Every hospital is scored on the same 26 risk-adjusted outcome measures CMS uses in its star ratings (mortality, readmissions, and safety). These were standardized and combined into a single Outcomes score. That score is paired with Per-Episode Cost (risk-adjusted Medicare spending per care episode, which is the total spend underlying CMS's MSPB measure) and split at the medians into four value tiers. There are 2,306 hospitals in this final dataset.
2. Hospital_Analysis.ipynb contains the statistical work, including an independent-samples t-test, one-way ANOVA with Tukey comparisons, two linear regressions (modeling outcomes and cost), correlations, the random-forest models for high value and cost, and the comparison against CMS star ratings.

A deliberate design choice I made was scoring outcomes on clinical measures only (mortality, readmissions, and safety) and leaving out the patient-experience and broader availability measures CMS folds into its stars. This is exactly the reason this lens diverges from the star ratings in interpretable ways.

<details>
<summary><b>Technical deep-dive: models, diagnostics, and regression detail</b> (click to expand)</summary>

<h3>The core result: cost is learnable, but clinical outcomes are not</h3>

<p>Two method families and two targets yield one consistent verdict:</p>

<table>
  <tr><th></th><th>Outcome target</th><th>Cost target</th></tr>
  <tr><td><b>Linear regression</b> (adj. R²)</td><td>0.13</td><td>0.38</td></tr>
  <tr><td><b>Random forest</b> (ROC-AUC)</td><td>0.65</td><td>0.77</td></tr>
</table>

<p>Linear models and tree-based models make very different assumptions. Both types of models in this analysis agree that outcomes are barely explained while cost is moderately explained. This indicates that the weakness is in the <em>data</em>, not the model. The patient experience and availability features capture information about how a hospital operates, which relates to cost, far better than they predict its clinical results.</p>

<h3>Regression coefficients</h3>

<img src="https://github.com/KabiraL/Hospital_Assessment_Databricks/blob/9e9ec77f594bcc481319414a84e8faca621f35a5/Data/outcomes_coef_plot.png" alt="What moves a hospital's clinical-outcome score" width="700">

<img src="https://github.com/KabiraL/Hospital_Assessment_Databricks/blob/9e9ec77f594bcc481319414a84e8faca621f35a5/Data/cost_coef_plot.png" alt="What moves a hospital's per-episode cost" width="700">

<p><em>Standardized OLS coefficients with 95% confidence intervals. The effects are real but, for outcomes, collectively modest (~13% of variation explained).</em></p>

<h3>Random forests: the first one gave a negative result, but the second one gave a usable one</h3>

<p>The first model tried to predict the high-value combination (high outcome <b>and</b> low cost) directly and couldn't. The model had about 42% precision on held-out data, capped by the weak outcome dimension. Rebuilding it to predict <b>low cost alone</b> succeeded. It achieved  approximately 84% precision at a 0.65 probability threshold on the locked test set. </p>

<img src="https://github.com/KabiraL/Hospital_Assessment_Databricks/blob/9e9ec77f594bcc481319414a84e8faca621f35a5/Data/cost_permutation_importance.png" alt="Low-cost model permutation importance" width="700">

<p><em>Permutation importance on held-out data: ED efficiency, size, and region carry the low-cost model.</em></p>

<h3>Why the high-value corner is hard by definition</h3>

<p>Outcome and cost are mildly <em>positively</em> correlated. The regression coefficient, the value-tier counts, and the model's difficulty all agree. This suggests that "high outcome / low cost" hospitals fight the prevailing grain. They are genuine efficiency outliers. You can't reliably spot exceptions from coarse proxies. The reliable way to find them is to measure them directly.</p>

<h3>Benchmark check against CMS stars</h3>

<p>The value flag agrees with the official 1–5 star ratings on ~70% of hospitals (n = 2,306 with complete data). The ~30% divergence is concentrated exactly where the two lenses <em>should</em> differ, which is the cost dimension the stars ignore. This finding validates the outcome scoring while showing what the cost-inclusion measure adds.</p>

<h3>Repository structure</h3>

```
Hospital_Assessment_Databricks/
├── Hospital Data SQL Analysis.ipynb
├── Hospital_Analysis.ipynb
├── Data/                                  # cleaned datasets, images, data dictionary
├── Hospital_Value_Executive_Deck.pdf      # executive summary deck
└── README.md
```
<h3>Tech stack</h3>

<p><b>SQL</b> — data cleaning, variable creation, and data manipulation &middot; <b>Databricks</b> — data management and visualizations &middot;<b>Python</b> — pandas, NumPy, statsmodels, scikit-learn, matplotlib, seaborn &middot; <b>Tableau</b> — interactive value map</p>

### Data

Seven public datasets from the [CMS Provider Data Catalog — Hospitals](https://data.cms.gov/provider-data/topics/hospitals), covering ~5,000 hospitals:

- [Complications & deaths (95,780 observations)](https://data.cms.gov/provider-data/dataset/ynj2-r877)
- [Unplanned hospital visits (67,046 observations)](https://data.cms.gov/provider-data/dataset/632h-zaca)
- [Healthcare-associated infections (172,404 observations)](https://data.cms.gov/provider-data/dataset/77hc-ibv8)
- [Medicare hospital spending by claim (63,646 observations)](https://data.cms.gov/provider-data/dataset/nrth-mfg3)
- [HCAHPS patient survey (325,652 observations)](https://data.cms.gov/provider-data/dataset/dgck-syfz)
- [Timely & effective care (138,129 observations)](https://data.cms.gov/provider-data/dataset/yv7e-xc69)
- [General hospital information (5,426 observations)](https://data.cms.gov/provider-data/dataset/xubh-q36u)

**[View the data dictionary →](https://github.com/KabiraL/Hospital_Assessment_Databricks/blob/9e9ec77f594bcc481319414a84e8faca621f35a5/Data/Data%20Dictionary%20of%20Predictors.pdf)**


<img width="900" alt="CMS data model" src="https://github.com/KabiraL/Hospital_Assessment_Databricks/blob/7facaa4af08acb15961addb765f4dbff49c4f316/Data/TableStructure.png" />

The seven CMS source datasets and the fields drawn from each.

### Issues

There were data quality issues with 8 instances of Facility IDs having two completely different hospitals assigned to each Facility ID in the General hospital information dataset (~0.2% of hospitals). This error was discovered late in the analysis, as the original Python code had silently dropped the duplicate. At that scale the medians and model metrics don't move, so I chose to document the error and left the records as-is rather than re-run the analysis end-to-end.

### If there was more time ...

- I would add additional predictors from the Timely & effective care dataset to see if they contributed to strengthening the models.
- I would also add more predictors from Complications & deaths, Unplanned hospital visits, and Healthcare-associated infections datasets to the Outcomes measure to see if it materially shifted.
- All potentially-relevant predictors were used from the HCAHPS patient survey and General hospital information datasets.
- It could also be interesting to explore other CMS datasets to see if any of them hold potential future predictors.

</details>

By Leah Kabira · [LinkedIn](https://linkedin.com/in/leahkabira)
