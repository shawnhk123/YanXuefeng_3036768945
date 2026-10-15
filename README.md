## YanXuefeng_3036768945
Card &amp; Krueger(1994) minimum wage replication. DID design replicating Tables2-4 &amp; Figure1. Core findings match original; minor deviations from sample rules. Includes ownership heterogeneity extension, raw data, Python code, outputs and paper reference.
## Core Replication

- Replicated outputs: Tables 2–4 and Figure 1 from the original paper
- Key findings: Core conclusions align with the original study. The minimum wage hike did not reduce fast-food employment; instead, full-time-equivalent (FTE) employment in New Jersey increased slightly relative to Pennsylvania.

## Independent Extension

Heterogeneity analysis by store ownership type (company-owned vs. franchisee-owned):

- The positive employment effect of the minimum wage is statistically significant only for company-owned stores.
- The interaction term between policy treatment and ownership type is statistically insignificant. There is no evidence of heterogeneous policy effects across the two groups.

## Repository Contents

- `data/`: Raw survey dataset `public.csv`
- `code/`: Full Python Jupyter Notebook covering data preprocessing, replication of all tables and figures, and extension analysis
- `output/`: Replicated Tables 2–4, Figure 1, and results of the independent extension
- `reference/`: Original paper for result comparison

## How to Run

Open the Jupyter Notebook in the `code/` folder and run all cells. All outputs will be generated directly from the raw input data.
