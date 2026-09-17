# Predicting 180-Day Mortality in Critically Ill Hospitalized Patients (SUPPORT2)

## 1. Background and research question

This project uses the SUPPORT2 dataset. It holds data on 9,105 critically ill patients, hospitalized across five U.S. medical centers between 1989 and 1994. Patients entered the study with one of nine serious illness categories. These include acute respiratory failure, COPD, congestive heart failure, liver disease, coma, colon cancer, lung cancer, and two forms of multi-organ system failure.

The main research question is this. Using demographic, diagnostic, and physiological information available by day 3 of hospitalization, can we predict whether a patient dies within 180 days of study entry, and which of these baseline factors matter most.

A 180-day horizon fits this project for two reasons. It matches the endpoint the original SUPPORT study itself was built around. And it works cleanly with this data. Every patient who survived was followed for at least 344 days, so no one had to be dropped for being censored before we knew their 180-day outcome. This gives a usable sample of all 9,105 patients, with a fairly balanced split. 46.8 percent died within 180 days.

A second, related question uses the full survival-time structure of the data, `death` and `d.time` together, instead of the fixed 180-day cutoff. This shows how risk builds up over time, not just at one single point.

This project reflects a real motivation behind the original SUPPORT study. Doctors treating seriously ill patients often have to make hard decisions about how aggressively to keep treating someone. A reasonable, data-based estimate of mortality risk can support these conversations. It does not replace a doctor's judgment, and nothing in this report should be read as a causal claim. This is observational hospital data, not an experiment.

## 2. Data preparation

### Variables that were excluded

Several variables in the raw data were removed before modeling. Some leak information from after the point we would want to predict from. Others get explicitly flagged by the dataset's own documentation as unsuitable predictors.

`surv2m` and `surv6m`, the original SUPPORT model's own survival probability estimate, along with `prg2m` and `prg6m`, the treating physician's survival estimate, are already predictions of the outcome. Using them would mean copying someone else's answer instead of making our own.

`aps` and `sps` are severity scores built from the same raw physiology values already sitting in the dataset as separate columns. Including them means partly re-learning someone else's summary of the data, instead of learning from the raw numbers directly.

`avtisst` covers the average TISS score across days 3 to 25, according to the documentation, a much later window than our day-3 prediction point.

`dnr` and `dnrday` looked fine at first, but checking the data directly showed something important. For about 65 percent of patients who never had a DNR order, `dnrday` gets set to exactly equal length of stay. It secretly encodes an outcome-related number, not a real baseline fact.

`sfdm2` is a functional-status score taken at a 2-month follow-up interview. One of its five categories reads literally as patient died before 2 months, so part of this variable is simply the outcome under a different name.

`charges`, `totcst`, `totmcst`, and `slos` are all totals that stay unknown until the hospital stay ends.

`dzclass`, a broader version of the more useful `dzgroup`, and `adlp` and `adls`, two raw functional-status measures with a lot of missing data, got dropped for a different reason. They are redundant rather than leaky. The dataset already carries a complete, calibrated version of this information in `adlsc`.

After removing these columns, 27 predictors remain. They cover demographics such as age, sex, race, education, and income, diagnosis information such as disease group, cancer status, comorbidity count, diabetes, and dementia, and physiological and functional measurements taken on day 3.

### Missing data

The missing values in this dataset are not random. Checking missingness against disease group makes this clear. A respiratory function test, `pafi`, was missing in only 14 percent of COPD patients but in 63 percent of Colon Cancer patients, who mostly would not need that test. `adlp`, which requires directly interviewing the patient, was missing in 99 percent of Coma patients, for an obvious reason. You cannot interview someone who is unconscious. In a lot of cases, a value goes missing because the test was not clinically needed, or because the patient's condition made it impossible to collect.

For nine physiology labs, the dataset's own documentation gives recommended normal values to fill in for missing entries. Albumin gets 3.5, creatinine gets 1.01, and so on. This rests on the idea that if a doctor did not order a test, they probably did not suspect anything was wrong. These are fixed numbers, not values calculated from our own data, so using them creates no leakage, no matter how the data gets split later. Two labs, `glucose` and `ph`, did not have an official fill-in value listed, so standard clinical reference numbers were used instead, following the same logic. This is a weaker assumption than the other seven, since the dataset's own documentation does not actually recommend it.

For `income`, a categorical and financial variable, a separate missing category was created instead of guessing a value. For `edu` and a few columns with only 1 or 2 missing rows, the median was used, but only calculated from the training data, once the data was actually split, so no information from the test set leaks into the training process.

A later check, in Section 10, found that switching from these normal fill-in values to simple median imputation made no real difference to model performance. The test AUC came out exactly the same either way. The more careful approach still stands as the better justified one, but the final results here do not depend heavily on it.

## 3. Exploratory findings

Mortality risk changes a great deal depending on disease group, far more than almost any other variable in this dataset. 180-day mortality ranges from 27 percent in CHF and 32 percent in COPD up to 63 percent in Lung Cancer, 75 percent in MOSF with Malignancy, and 75.5 percent in Coma. This order matches what you would expect clinically, and it is not random. Age matters too, but the effect is much gentler, rising from about 26 percent in patients under 20 to 55 to 60 percent in patients over 80. That makes sense, since everyone in this dataset already counts as critically ill.

## 4. Statistical testing, a significant result is not always an important one

Three variables were tested formally against the 180-day outcome. Each represents a different kind of data and a different strength of relationship.

Age, tested with Welch's t-test, shows a mean difference of 3.38 years between patients who died and patients who survived, with a 95 percent confidence interval of 2.74 to 4.02 and a p-value under 0.0001. But the effect size, Cohen's d, comes out at only 0.218, which counts as small.

Disease group, tested with a chi-square test, shows a stronger, moderate-sized effect. Cramer's V reaches 0.303, with a chi-square statistic of 838.1 and a p-value close to zero, matching the large spread in mortality by group seen above.

Creatinine, tested with a Mann-Whitney U test because its distribution is heavily skewed, came back statistically significant at p equals 6.5e-7. But the median value sat exactly the same in both groups, at 1.2000. The effect size, a rank-biserial correlation, landed at negative 0.06, close to zero. With over 9,000 patients, even very small, unimportant differences can turn statistically significant, and this stands as a clear example. It does not mean creatinine turns useless once combined with other variables in a full model, but on its own, this result should not carry much weight.

## 5. Survival analysis

Using the Kaplan-Meier method on the full cohort, the median survival time comes out at 233 days. Survival probability sits at 73 percent at 30 days, 61 percent at 90 days, 53 percent at 180 days, matching the 46.8 percent mortality figure above, 45 percent at one year, and 36 percent at two years.

Survival also looks quite different across disease groups, both in overall level and in shape. CHF patients hold the strongest numbers among the groups checked closely here. 91 percent are alive at 30 days, 81 percent at 90 days, 73 percent at 180 days, 62 percent at one year, and 48 percent at two years. Patients with ARF or MOSF and sepsis sit lower, at 71 percent, 61 percent, 55 percent, 50 percent, and 44 percent across those same points. Lung Cancer patients start reasonably high, at 76 percent survival by 30 days, but drop faster than the other groups, falling to 54 percent at 90 days, 37 percent at 180 days, 21 percent at one year, and just 10 percent at two years. Coma patients start lowest of all, at 35 percent survival by 30 days, and the curve flattens after that, ending at 28 percent, 25 percent, 22 percent, and 20 percent across the remaining points.

A log-rank test confirms these differences are real rather than random noise, with a chi-square statistic of 1222.4 and a p-value close to zero. A Cox proportional hazards model was fit using a core set of clinically meaningful predictors. It was stratified by disease group, because the curves above are not just shifted versions of each other. Coma drops sharply early and then flattens out, while Lung Cancer declines steadily the whole time. A single model without stratification cannot represent both patterns well.

Stratifying fixed the assumption violations that came from disease group. But six other variables, age, comorbidity count, blood pressure, bilirubin, PaO2/FiO2 ratio, and coma score, still show some violation of the proportional-hazards assumption, even after this fix. Their hazard ratios should be read as an average effect across the roughly 5.5-year follow-up period, not as a fixed multiplier that stays constant at every point in time.

A few hazard ratios from this model stand out, each with a 95 percent confidence interval attached. Age comes in at 1.015 per year, with a range of 1.013 to 1.017. Coma score sits at 1.014 per point, ranging from 1.013 to 1.016. Comorbidity count reaches 1.098 per comorbidity, ranging from 1.075 to 1.122. Functional status, measured as `adlsc`, lands at 1.087 per point, ranging from 1.073 to 1.100. Bilirubin and creatinine sit at 1.017 and 1.023 per unit, meaning worse liver or kidney function links to slightly higher hazard. Male sex reaches 1.117, ranging from 1.062 to 1.176. Blood pressure, oxygenation, and sodium each turned out mildly protective as they rose. Albumin did not reach significance in this model, with a p-value of 0.657, most likely because its effect overlaps with the other severity markers already included.

## 6. Machine learning models

A leakage-safe pipeline handled this part. All imputation, scaling, and encoding got refit again inside each cross-validation fold, and a held-out test set, 20 percent of the data, stayed untouched until the final evaluation. Three models were compared against a dummy baseline on the 180-day classification target.

The dummy model, which just predicts the overall base rate, scores an AUC of 0.500, exactly as expected. Logistic Regression starts at 0.754 with a standard deviation of 0.016 across the five folds, and tuning barely moves it, staying at 0.754 with a regularization strength of C equals 0.1. Random Forest starts at 0.760 with a standard deviation of 0.015 and improves slightly to 0.761 after tuning. Gradient Boosting starts lowest of the three real models, at 0.754 with a standard deviation of 0.011, and improves to 0.758 after tuning.

Without tuning, a paired comparison using the same cross-validation folds found no statistically significant difference between any of these three models. Random Forest against Logistic Regression gave a p-value of 0.132, and Gradient Boosting against Logistic Regression gave 0.971. Sensible hyperparameter tuning did not change this picture much either.

One important check came out of this stage. Comparing each tuned model's own training-set score against its cross-validation score showed a real gap between the models. Logistic Regression barely overfits at all, with a gap of only 0.006. Random Forest shows a much larger gap, 0.156, and Gradient Boosting a smaller but still noticeable one, 0.106. A model that does much better on the data it trained on than on new, held-out data has partly memorized specific patients rather than learned something that generalizes well, even when its cross-validation score looks fine on its own.

## 7. Final evaluation

On the untouched 20 percent test set, 1,821 patients, Logistic Regression reached an AUC of 0.755, with a 95 percent confidence interval of 0.733 to 0.777 from bootstrapping, very close to its cross-validation estimate. Random Forest scored higher on this particular test set, at 0.773. A paired bootstrap test confirmed this gap held up consistently, at a difference of 0.018 on this test set, with a 95 percent confidence interval of 0.009 to 0.029, a tighter result than the earlier cross-validation comparison suggested. Section 10 explains why these two checks disagreed.

Calibration told a different story than raw discrimination. Logistic Regression's predicted probabilities matched the observed outcome rates closely, across almost every decile. Random Forest performed fine through the middle of the risk range, but consistently underestimated risk in the top three deciles. It predicted 0.602, 0.685, and 0.790, when the actual observed mortality rates in those groups reached 0.725, 0.786, and 0.874. This is a known weakness of random forests. Averaging over many trees pulls probability estimates toward the middle, and it matters most for exactly the patients this kind of tool should serve best, the highest-risk ones. Both models still beat a no-skill baseline clearly on the Brier score, 0.200 and 0.196, compared to 0.249 for a model that just guesses the overall base rate for every patient.

At a standard 0.5 probability threshold, Logistic Regression reaches a sensitivity of 0.617, a specificity of 0.755, a PPV of 0.689, and an NPV of 0.691. About 38 percent of patients who actually die within 180 days would get missed at this cutoff. Since the intended use here supports early conversations rather than gatekeeping treatment, a lower threshold that trades some specificity for more sensitivity would probably suit this purpose better. The exact right threshold belongs to whoever uses this tool, not something a model decides on its own.

## 8. What drives the predictions

Logistic Regression's coefficients and Random Forest's permutation importance, a more trustworthy option than the model's built-in importance measure, which tends to favor variables with many categories, agree closely on what matters most. Disease group and cancer status, coma score, functional status, and age all rank near the top for both models, even though they represent very different kinds of algorithms. This agreement across two different approaches suggests the pattern is real, not something specific to one model.

A couple of findings deserve more explanation. The Coma disease-group category carries a smaller coefficient, an odds ratio of 1.24, than its very high raw mortality rate of 75.5 percent would suggest. The coma severity score already sits in the model separately, and it captures most of that effect on its own. Once you already know a patient's coma score, the diagnostic label itself adds less extra information. A day-of-admission variable, `hday`, also turned out to matter more than expected, in both models. The most likely explanation holds that a longer gap before study entry reflects a more complicated hospital course, but the data alone cannot confirm this with certainty, so it deserves some caution.

Income and race both showed small effects in the Logistic Regression coefficients, income above $50k at an odds ratio of 0.83 and Hispanic race also at 0.83, and neither made it into Random Forest's list of top predictors either. Both models agree these are minor factors, not major drivers. These associations most likely reflect real-world patterns such as unequal access to healthcare, rather than anything biological, and should not be treated as a general rule beyond this one dataset.

## 9. Where the model gets it wrong

Looking closely at the test set's most confident mistakes, 31 high-confidence false positives and 35 high-confidence false negatives out of 1,821 patients, shows two clear and opposite patterns.

High-confidence false positives, patients the model felt very sure would die but who survived, cluster heavily in MOSF with Malignancy, 39 percent of this error group against 8 percent of the overall test set, and Coma, 19 percent against 6 percent. These patients also carried notably higher coma scores and creatinine levels than average. They genuinely looked very sick when measured, and the model correctly picked up on that. But some of them recovered anyway. A single measurement at baseline cannot predict how well a patient responds to treatment afterward.

High-confidence false negatives show close to the opposite pattern. 51 percent are CHF patients, against 15 percent overall, and 23 percent are COPD patients, against 11 percent overall, with lower coma scores, better functional status, and younger ages than average. These patients looked stable at baseline, and died anyway within 180 days. CHF and COPD count as chronic conditions that can look stable on a given day and then suddenly worsen. No baseline snapshot, no matter which model reads it, can see that coming. This suggests a real limit of relying on a single point in time, and it applies regardless of which model gets used.

## 10. Robustness checks

Two open questions from earlier in the analysis got tested directly, instead of staying unresolved.

The first question asked whether Random Forest's edge over Logistic Regression held up as real. The train and test split got repeated 10 times, using different random seeds but the same tuned hyperparameters each time. Random Forest beat Logistic Regression in all 10 splits, with a small but very consistent difference, a mean AUC difference of 0.011 and a standard deviation of only 0.004. This actually explains the earlier disagreement between the cross-validation comparison, which found no significant difference, and the single test-set comparison, which did. The cross-validation test was not wrong. It simply lacked enough statistical power, with only 5 folds, to detect a real but small effect. Random Forest does carry a genuine, if modest, advantage in discrimination. This does not settle which model counts as better overall, though. It still shows worse calibration for the highest-risk patients, and clearer signs of overfitting, so the right choice depends on whether pure ranking ability matters more, or a trustworthy probability estimate does.

The second question asked whether the missing-data strategy actually mattered. As mentioned in Section 2, replacing the documented clinical fill-in values with simple median imputation left both performance and the model's coefficients almost unchanged. The final conclusions here do not hinge on that particular decision.

## 11. Limitations

The `glucose` and `ph` fill-in values used for imputation extend the dataset's documented approach in a reasonable way, but they are not officially documented for this dataset, unlike the other seven physiology labs.

Six variables in the Cox survival model still show some violation of the proportional-hazards assumption, even after stratifying by disease group. Their hazard ratios should be read as average effects across the follow-up period, not as fixed constants.

Random Forest consistently underestimates risk for the highest-risk patients, exactly the group a clinical risk tool should get most accurate. This counts as a real argument against using it as the main model, even with its small, statistically consistent advantage in AUC.

The reason `hday` matters as much as it does is not fully understood. This report offers a plausible explanation, but it remains unconfirmed.

Income and race show small associations with the outcome that most likely reflect social and healthcare-access patterns, rather than anything biological. These should not get generalized beyond this particular dataset.

Both the machine-learning and survival models rely on a single snapshot of the patient, taken on day 3. The error analysis showed this consistently underestimates risk for CHF and COPD patients who later get worse, and overestimates risk for the sickest-looking patients who recover. These limitations come from the design of the prediction task itself, not from any particular model.

This is observational data from five U.S. hospitals between 1989 and 1994. None of these findings prove causation, and they may not generalize well to different patient populations, different healthcare systems, or more recent medical practice.

## 12. Conclusion

Baseline demographic, diagnostic, and day-3 physiological data can predict 180-day mortality in this population, but a clear ceiling sits around AUC 0.75 to 0.76. This ceiling stayed roughly the same across three different types of model and a range of tuning attempts, which suggests it reflects a genuine limit on what a single early snapshot can predict, rather than a weakness in the modeling approach. Disease group, cancer status, coma score, functional status, and age come up consistently as the main drivers of risk, across statistical testing, survival analysis, and two very different machine learning models. Where the model fails also tells a consistent, clinically sensible story. It overestimates risk for very sick-looking patients who go on to recover, and underestimates risk for stable-looking CHF and COPD patients who decline later. This pattern comes from the limits of predicting from a single point in time, not from any specific algorithm's weaknesses.
