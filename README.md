# Predicting 180-Day Mortality in Critically Ill Patients (SUPPORT2)

This project uses the SUPPORT2 dataset, which holds data on 9,105 critically ill patients from five U.S. hospitals. The main question is this. Using information available by day 3 of a hospital stay, can we predict whether a patient dies within the next 180 days, and which factors matter most for that prediction.

## Key results

Three different models, Logistic Regression, Random Forest, and Gradient Boosting, land on roughly the same result on the test set, an AUC around 0.75 to 0.76. That stayed true even after tuning. This points to a real limit on what a single baseline snapshot can predict, not a weakness in any one model.

Random Forest ranks patients by risk slightly better than Logistic Regression, and this held up across ten repeated train and test splits, so the difference looks genuine rather than luck. But Random Forest has a real problem too. It underestimates risk for the patients who are actually highest risk, exactly the group this kind of tool needs to get right. It also shows more signs of overfitting. Logistic Regression stays the better choice when the goal is a trustworthy probability rather than just a ranking.

Disease group, cancer status, coma severity, functional status, and age come up as the strongest factors again and again, in the statistical tests, in the survival model, and in both machine learning models.

Looking closely at the model's biggest mistakes shows a clear pattern. It tends to overestimate risk for patients who look very sick at first but then recover. And it underestimates risk for CHF and COPD patients who look stable at first but get worse later. This comes from relying on one snapshot in time, not from any specific model.

The full report, with all the numbers and the reasoning behind each decision, sits in [`reports/final_report.md`](reports/final_report.md).

## Repository structure

```
├── data/
│   └── support2.csv          raw dataset (see Data source below)
├── notebooks/
│   └── SUPPORT2_analysis.ipynb   full analysis, runs top to bottom
├── reports/
│   └── final_report.md       full written report
├── requirements.txt
└── README.md
```

## How to run this

```bash
pip install -r requirements.txt
jupyter notebook notebooks/SUPPORT2_analysis.ipynb
```

It takes a few minutes to run start to finish, and does not need a GPU. Make sure `support2.csv` sits in the same folder you run it from, or change the file path in the first code cell if you keep it inside `data/`.

## Data source

The SUPPORT2 dataset comes from the Study to Understand Prognoses and Preferences for Outcomes and Risks of Treatments. Vanderbilt University's Department of Biostatistics makes it public at https://hbiostat.org/data/repo/supportdesc

## Limitations

This is old, observational data from 1989 to 1994, so none of the results here prove that one thing causes another. The final report explains the model's two clearest weak points in more depth, its overconfidence about high-severity patients who recover, and its underconfidence about CHF and COPD patients who decline later, along with a few other limitations worth reading before drawing conclusions from this work.
