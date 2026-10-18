# AI-Use Disclosure 
**Project:** Replication of Card & Krueger (1994), "Minimum Wages and Employment: A Case Study of the Fast-Food Industry in New Jersey and Pennsylvania"
**Deliverables covered:** Table 2, Table 3, Table 4, Figure 1, the Independent Extension, and the accompanying report text.

## 1. Where and how AI was used 

 An AI coding assistant (DeepSeek, accessed over the internet through the AI extension embedded in Visual Studio Code) was used as a coding and drafting aid throughout the project. Its use fell into three areas. First, it wrote the Python/pandas code that reproduces the paper: the full-time-equivalent (FTE) construction, the treatment of permanently and temporarily closed stores, the means and two-sample tests in Table 2, the grouped means and difference-in-differences estimates in Table 3, the reduced-form regressions in Table 4, and the starting-wage histogram in Figure 1. Second, it wrote the estimation code for the independent extension on ownership heterogeneity, and the export routines that write the results to Word and Excel. 

The AI never had direct access to the dataset. All data loading, cleaning and execution took place locally, in my own Jupyter kernel inside VS Code; the AI worked only from the codebook, the paper's published tables and figure screenshots, and the console output that I pasted back to it. Because the local environment was minimal (SciPy, for example, was not installed), the AI wrote the estimator and the p-value routines directly in NumPy rather than relying on standard statistical packages.

## 2. How AI-generated code, analysis, and writing were checked 

 Every number produced by the AI was verified against the published results before it was accepted.

- **Cell-by-cell comparison.** Each reproduced table was compared with the paper line by line: means, standard errors, t-statistics and sample sizes. Sample sizes were additionally cross-checked against independent counts taken from the raw data (410 stores in total, 79 in Pennsylvania and 331 in New Jersey; 314 and 76 wave-1 starting-wage observations; wage-group counts of 101 / 140 / 73).
- **Formulas taken from the paper, not from the AI.** The FTE definition (managers + full-time workers + 0.5 × part-time workers) and the closure rules were re-read in the paper's text and footnotes before any code was written against them. Where a printed figure could not be reproduced from a formula stated in the paper, the discrepancy was required to be reported as a documented limitation rather than closed by fitting an arbitrary constant.
- **Internal consistency checks.** Identities such as "row 3 mean = row 2 mean − row 1 mean" were used as independent tests of the code.
- **Text checked against the numbers actually produced.** The prose was read against the verified output line by line, and any figure that had not been confirmed was deleted. For example, R², the standard error of the regression and the region-dummy coefficients are not cited in the Independent Extension section, because those values were truncated in the notebook display and never verified.

## 3. Important AI errors or suggestions that I corrected or rejected 


1. **Table 2, "percentage of full-time employees" — rejected a fabricated correction.** The AI's first two formulas produced 28.6 / 30.3 and 37.7 / 38.8 against the published 32.8 / 35.0. It then proposed adding a fitted constant to close the gap. I rejected this outright as inventing a formula that the paper does not contain, and required the row to be reported with an explicit documented caveat instead.
2. **Table 3, row 3 standard errors — corrected a variance error.** The AI's first calculation produced 1.23 / 0.47 / 0.82 / 0.68 / 0.99 against the published 1.25 / 0.54 / 0.95 / 0.84 / 1.14. The error was that the two means in that row are computed on the same, overlapping stores, so the variance of their difference must include the covariance term. The corrected version reproduces the published row.
3. **Table 4 sample — a mismatch that was reported, not hidden.** The AI's first sample restriction also required a wave-2 starting wage, producing 351 stores rather than the paper's 357. This could not be resolved from the public-use extract; it is disclosed as a limitation of the replication rather than concealed.
4. **An unverified explanation — flagged and removed.** In drafting the Core Replication section, the AI presented an explanation for the six-store difference as though it had been established, although the diagnostic supporting it had never been run. I identified this and required the claim to be marked as a hypothesis rather than a finding.
5. **Practical errors in the export routines.** The AI's export code did not state that `python-docx` and `openpyxl` had to be installed; both produced `ModuleNotFoundError`, which I resolved by installing the packages. Its first p-value implementation also assumed SciPy was available; since it was not, the routine was rewritten in NumPy.

## Declaration 
All analytical decisions, all verification of results against the original paper, and all final text in the submission are my own. The AI was used as an implementation and drafting tool, in the manner described above.