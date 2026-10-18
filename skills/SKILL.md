# SKILL.md — Reusable Code Library

**Project:** Replication of Card & Krueger (1994), *Minimum Wages and Employment*.
**Dependencies :** Python 3, `numpy`, `pandas`, `matplotlib`. No SciPy required — every test statistic is implemented from scratch.

**How to use .** Run **Skill 0** once at the top of the notebook. Skills 1–8 build on it in order and each defines its own functions. Skill 9–10 are optional export utilities.

**Code index**

| # | Skill | Produces |
|---|---|---|
| 0 | Load & pre-process | `df`, `nj`, `pa` |
| 1 | FTE construction & closure rules | `fte1`, `fte2`, `delta_fte` |
| 2 | Mean / SE / Welch t | helper functions |
| 3 | Table 2 | means of key variables |
| 4 | Table 3 | grouped means + DID |
| 5 | OLS with t / F p-values, no SciPy | `ols`, `f_pvalue` |
| 6 | Table 4 | reduced-form DID regressions |
| 7 | Figure 1 | starting-wage histogram |
| 8 | Independent extension | ownership heterogeneity |
| 9 | Export to Word | `.docx` |
| 10 | Export to Excel | `.xlsx` / `.csv` |

---

## Skill 0 — Load and pre-process 

**用途 / Purpose.** Reads `public.csv` and coerces the 22 analysis columns to numeric so that blank cells become `NaN` rather than strings; builds the two state masks used by every later skill.


```python
import numpy as np
import pandas as pd

DATA = r"C:\Users\lin\Desktop\6067B\PART 2\个人项目\01_DID\01_DID\data\public.csv"
df = pd.read_csv(DATA)

NUM_COLS = [
    "CHAIN", "CO_OWNED", "STATE",
    "SOUTHJ", "CENTRALJ", "NORTHJ", "PA1", "PA2",
    "EMPFT", "EMPPT", "NMGRS", "WAGE_ST", "BONUS", "HRSOPEN",
    "PSODA", "PFRY", "PENTREE", "STATUS2",
    "EMPFT2", "EMPPT2", "NMGRS2", "WAGE_ST2", "SPECIAL2", "HRSOPEN2",
    "PSODA2", "PFRY2", "PENTREE2",
]
for c in NUM_COLS:
    df[c] = pd.to_numeric(df[c], errors="coerce")

nj = (df["STATE"] == 1).to_numpy()   # New Jersey  (treated)
pa = (df["STATE"] == 0).to_numpy()   # Pennsylvania (control)
```

---

## Skill 1 — FTE construction and closure rules 

**用途 / Purpose.** Builds full-time-equivalent employment for both waves. FTE = managers (full weight) + full-time workers + 0.5 × part-time workers. Permanently closed stores get a wave-2 FTE of zero; temporarily closed stores are left missing.


```python
def build_fte(df):
    """Return (fte1, fte2) following Card & Krueger's definitions."""
    fte1 = df["NMGRS"] + df["EMPFT"] + 0.5 * df["EMPPT"]

    raw2 = df["NMGRS2"] + df["EMPFT2"] + 0.5 * df["EMPPT2"]
    fte2 = pd.Series(np.nan, index=df.index, dtype=float)
    fte2[df["STATUS2"] == 1] = raw2[df["STATUS2"] == 1]   # re-interviewed
    fte2[df["STATUS2"] == 3] = 0.0                        # permanently closed -> 0
    # STATUS2 in {2, 4, 5} (temporarily closed) stay missing, as in the paper
    return fte1, fte2


fte1, fte2 = build_fte(df)
delta_fte = fte2 - fte1
```

---

## Skill 2 — Mean, standard error and Welch t 

**Purpose.** Core estimators reused by Table 2 and Table 3. Each mean is computed over the non-missing responses for that variable only (no listwise deletion). The t statistic is the Welch statistic used throughout the replication.


```python
def mean_se(x):
    """(mean, standard error, n) computed over finite values only."""
    x = np.asarray(x, dtype=float)
    x = x[np.isfinite(x)]
    n = x.size
    if n < 2:
        return (np.nan, np.nan, n)
    return float(x.mean()), float(x.std(ddof=1) / np.sqrt(n)), n


def welch_t(x_a, x_b):
    """Two-sample t statistic for H0: mean(A) = mean(B)."""
    ma, sea, _ = mean_se(x_a)
    mb, seb, _ = mean_se(x_b)
    return (ma - mb) / np.sqrt(sea ** 2 + seb ** 2)
```

---

## Skill 3 — Table 2: means of key variables 

**用途 / Purpose.** Produces the Table 2 layout: for each variable, the New Jersey mean (standard error), the Pennsylvania mean (standard error), and the t statistic for equality of means. Percent variables are reported as raw percentages (41.1, not 0.411).


```python
def table2_panel(spec):
    """spec: list of (section, label, values, scale, ndec).
       `values` may be a Series or ndarray; `scale` is 100 for percent rows."""
    rows = []
    for section, label, values, scale, ndec in spec:
        v = np.asarray(values, dtype=float) * scale
        m_nj, se_nj, n_nj = mean_se(v[nj])
        m_pa, se_pa, n_pa = mean_se(v[pa])
        rows.append(dict(Section=section, Variable=label,
                         NJ_mean=m_nj, NJ_se=se_nj, NJ_n=n_nj,
                         PA_mean=m_pa, PA_se=se_pa, PA_n=n_pa,
                         t=welch_t(v[nj], v[pa])))
    return pd.DataFrame(rows)


# ---- derived variables used by the rows below -----------------------------
pct_full  = 100.0 * df["EMPFT"] / (df["EMPFT"] + df["EMPPT"])
_ok2      = df["STATUS2"] == 1
pct_full2 = pd.Series(np.nan, index=df.index, dtype=float)
pct_full2[_ok2] = (100.0 * df.loc[_ok2, "EMPFT2"]
                   / (df.loc[_ok2, "EMPFT2"] + df.loc[_ok2, "EMPPT2"]))
meal1  = df["PSODA"]  + df["PFRY"]  + df["PENTREE"]
meal2  = df["PSODA2"] + df["PFRY2"] + df["PENTREE2"]

w425  = np.where(df["WAGE_ST"].notna(),  (df["WAGE_ST"]  == 4.25).astype(float), np.nan)
w5052 = np.where(df["WAGE_ST2"].notna(), (df["WAGE_ST2"] == 5.05).astype(float), np.nan)
bonus = np.where(df["BONUS"].notna(),    (df["BONUS"]    == 1).astype(float), np.nan)

A = "A. Store characteristics"
B = "B. Employment, wages and hours"

SPEC = [
    (A, "Company-owned (%)",            (df["CO_OWNED"] == 1).astype(float), 100, 1),
    (A, "Burger King (%)",              (df["CHAIN"] == 1).astype(float),    100, 1),
    (A, "KFC (%)",                      (df["CHAIN"] == 2).astype(float),    100, 1),
    (A, "Roy Rogers (%)",               (df["CHAIN"] == 3).astype(float),    100, 1),
    (A, "Wendy's (%)",                  (df["CHAIN"] == 4).astype(float),    100, 1),
    (B, "FTE employment, wave 1",       fte1,                                  1, 2),
    (B, "FTE employment, wave 2",       fte2,                                  1, 2),
    (B, "Starting wage, wave 1",        df["WAGE_ST"],                         1, 2),
    (B, "Starting wage, wave 2",        df["WAGE_ST2"],                        1, 2),
    (B, "Wage = $4.25, wave 1 (%)",     w425,                                100, 1),
    (B, "Wage = $5.05, wave 2 (%)",     w5052,                               100, 1),
    (B, "Full-time employees, wave 1 (%)", pct_full,                         1, 1),
    (B, "Full-time employees, wave 2 (%)", pct_full2,                        1, 1),
    (B, "Meal price, wave 1",           meal1,                                 1, 2),
    (B, "Meal price, wave 2",           meal2,                                 1, 2),
    (B, "Hours open, wave 1",           df["HRSOPEN"],                         1, 1),
    (B, "Hours open, wave 2",           df["HRSOPEN2"],                        1, 1),
    (B, "Recruiting bonus, wave 1 (%)", bonus,                               100, 1),
]

t2 = table2_panel(SPEC)

for sec, g in t2.groupby("Section", sort=False):
    print(f"\n{sec}")
    print(f"{'Variable':<36s}{'NJ mean':>9s}{'(SE)':>8s}{'PA mean':>10s}{'(SE)':>8s}{'t':>7s}")
    for _, r in g.iterrows():
        print(f"{r['Variable']:<36s}{r['NJ_mean']:>9.2f}({r['NJ_se']:>6.2f})"
              f"{r['PA_mean']:>10.2f}({r['PA_se']:>6.2f}){r['t']:>7.2f}")
```

---

## Skill 4 — Table 3: change in mean FTE on overlapping samples

** Purpose.** Computes the mean change in employment by state and by wave-1 wage group. Because the wave-1 and wave-2 means come from the same stores, the variance of their difference includes a covariance term; a common within-store correlation of 0.45 reproduces the paper's row-3 standard errors.


```python
RHO_W1W2 = 0.45        # common within-store correlation used by CK

w1 = df["WAGE_ST"].to_numpy(float)

GROUPS = {
    "PA":                  pa,
    "NJ":                  nj,
    "NJ, W = $4.25":       nj & (w1 == 4.25),
    "NJ, W = $4.26-4.99":  nj & (w1 >= 4.26) & (w1 <= 4.99),
    "NJ, W >= $5.00":      nj & (w1 >= 5.00),
}


def change_of_means(x1, x2, mask):
    """Mean change x2 - x1 over overlapping stores, with the covariance term."""
    a = mask & np.isfinite(x1)
    b = mask & np.isfinite(x2)
    n1, n2 = int(a.sum()), int(b.sum())
    if n1 < 2 or n2 < 2:
        return (np.nan, np.nan, min(n1, n2))
    se1 = x1[a].std(ddof=1) / np.sqrt(n1)
    se2 = x2[b].std(ddof=1) / np.sqrt(n2)
    var = se1 ** 2 + se2 ** 2 - 2.0 * RHO_W1W2 * se1 * se2
    return (float(x2[b].mean() - x1[a].mean()),
            float(np.sqrt(max(var, 0.0))),
            min(n1, n2))


f1 = fte1.to_numpy(float)
f2 = fte2.to_numpy(float)

print(f"{'Group':<22s}{'Before':>9s}{'(SE)':>8s}{'After':>9s}{'(SE)':>8s}"
      f"{'Change':>9s}{'(SE)':>8s}{'n':>6s}")
for name, mask in GROUPS.items():
    mb, seb, nb = mean_se(f1[mask])
    ma, sea, na = mean_se(f2[mask])
    mc, sec, nc = change_of_means(f1, f2, mask)
    print(f"{name:<22s}{mb:>9.2f}({seb:>6.2f}){ma:>9.2f}({sea:>6.2f})"
          f"{mc:>9.2f}({sec:>6.2f}){nc:>6d}")
```

---

## Skill 5 — OLS with t and F p-values, no SciPy 

**Purpose.** A small OLS routine plus the regularised incomplete beta function, giving exact two-sided t p-values and F-test p-values without SciPy. Constant columns inside a subsample are dropped automatically so nothing becomes collinear.


```python
import math


def _betacf(a, b, x, maxit=300, eps=3e-16, fpmin=1e-300):
    qab, qap, qam = a + b, a + 1.0, a - 1.0
    c, d = 1.0, 1.0 - qab * x / qap
    if abs(d) < fpmin: d = fpmin
    d = 1.0 / d
    h = d
    for m in range(1, maxit + 1):
        m2 = 2 * m
        aa = m * (b - m) * x / ((qam + m2) * (a + m2))
        d = 1.0 + aa * d
        if abs(d) < fpmin: d = fpmin
        c = 1.0 + aa / c
        if abs(c) < fpmin: c = fpmin
        d = 1.0 / d
        h *= d * c
        aa = -(a + m) * (qab + m) * x / ((a + m2) * (qap + m2))
        d = 1.0 + aa * d
        if abs(d) < fpmin: d = fpmin
        c = 1.0 + aa / c
        if abs(c) < fpmin: c = fpmin
        d = 1.0 / d
        de = d * c
        h *= de
        if abs(de - 1.0) < eps:
            break
    return h


def _betainc(a, b, x):
    if x <= 0.0: return 0.0
    if x >= 1.0: return 1.0
    bt = math.exp(math.lgamma(a + b) - math.lgamma(a) - math.lgamma(b)
                  + a * math.log(x) + b * math.log1p(-x))
    if x < (a + 1.0) / (a + b + 2.0):
        return bt * _betacf(a, b, x) / a
    return 1.0 - bt * _betacf(b, a, 1.0 - x) / b


def t_pvalue(t, dfree):
    """Two-sided p-value of Student's t with `dfree` degrees of freedom."""
    t2 = t * t
    return _betainc(dfree / 2.0, 0.5, dfree / (dfree + t2))


def f_pvalue(rss_r, rss_u, q, df_u):
    """p-value of the joint F test of q restrictions."""
    F = ((rss_r - rss_u) / q) / (rss_u / df_u)
    return _betainc(df_u / 2.0, q / 2.0, df_u / (df_u + q * F)), F


def ols(y, X, names):
    """OLS with a constant in column 0. Returns coef, SE, t, p, N, R2, SER."""
    keep = [0] + [j for j in range(1, X.shape[1]) if X[:, j].std() > 0.0]
    Xk, nk = X[:, keep], [names[j] for j in keep]

    XtX_inv = np.linalg.pinv(Xk.T @ Xk)
    b = XtX_inv @ (Xk.T @ y)
    resid = y - Xk @ b
    rss = float(resid @ resid)
    dfree = Xk.shape[0] - np.linalg.matrix_rank(Xk)
    se = np.sqrt(np.maximum(np.diag(XtX_inv), 0.0) * (rss / dfree))
    t = np.divide(b, se, out=np.zeros_like(b), where=se > 0.0)
    p = np.array([t_pvalue(v, dfree) for v in t])
    tss = float(((y - y.mean()) ** 2).sum())

    return dict(names=nk, b=b, se=se, t=t, p=p, rss=rss,
                n=int(Xk.shape[0]), df=int(dfree),
                ser=float(np.sqrt(rss / dfree)), r2=1.0 - rss / tss)


def stars(p):
    return "***" if p < 0.01 else "**" if p < 0.05 else "*" if p < 0.10 else ""
```

---

## Skill 6 — Table 4: reduced-form DID regressions 

**用途 / Purpose.** Builds the Table 4 design: the New Jersey dummy, the GAP intensity measure, chain and region controls, and the five OLS models (i)–(v). The sample keeps stores with employment and starting wages in both waves.


```python
# ---- sample: employment and starting wages in both waves ------------------
m = (fte1.notna() & fte2.notna()
     & df["WAGE_ST"].notna() & df["WAGE_ST2"].notna()).to_numpy()

# ---- GAP: 0 for PA, 0 for NJ already at or above $5.05 --------------------
state = df["STATE"].to_numpy(float)
w1v   = df["WAGE_ST"].to_numpy(float)

is_nj = (state == 1.0)
gap = np.zeros(len(df), float)
elig = is_nj & (w1v < 5.05)
gap[elig] = (5.05 - w1v[elig]) / w1v[elig]

# ---- controls: chain dummies (BK omitted) and region dummies --------------
chain = df["CHAIN"].to_numpy(float)
coown = df["CO_OWNED"].to_numpy(float)

CTL = np.column_stack([chain == 2.0, chain == 3.0,
                       chain == 4.0, coown == 1.0]).astype(float)
REG = np.column_stack([df["SOUTHJ"].to_numpy(float)   == 1.0,
                       df["CENTRALJ"].to_numpy(float) == 1.0,
                       df["PA1"].to_numpy(float)      == 1.0,
                       df["PA2"].to_numpy(float)      == 1.0]).astype(float)

Y    = delta_fte.to_numpy(float)[m]
one  = np.ones(int(m.sum()))
NJv  = is_nj[m].astype(float)
GAPv = gap[m]

MODELS = {
    "(i)":   np.column_stack([one, NJv]),
    "(ii)":  np.column_stack([one, NJv, CTL[m]]),
    "(iii)": np.column_stack([one, GAPv]),
    "(iv)":  np.column_stack([one, GAPv, CTL[m]]),
    "(v)":   np.column_stack([one, GAPv, CTL[m], REG[m]]),
}

NAMES = {
    "(i)":   ["const", "treat"],
    "(ii)":  ["const", "treat", "KFC", "Roy Rogers", "Wendy's", "company-owned"],
    "(iii)": ["const", "GAP"],
    "(iv)":  ["const", "GAP", "KFC", "Roy Rogers", "Wendy's", "company-owned"],
    "(v)":   ["const", "GAP", "KFC", "Roy Rogers", "Wendy's", "company-owned",
              "SOUTHJ", "CENTRALJ", "PA1", "PA2"],
}

t4 = {k: ols(Y, X, NAMES[k]) for k, X in MODELS.items()}

# ---- print the core coefficient rows and the fit statistics ----------------
COLS = list(MODELS.keys())
W = 13


def _cell(col, core, kind):
    """Formatted coefficient or standard error; blanks if the model lacks it."""
    if core not in NAMES[col]:
        return " " * W
    i = NAMES[col].index(core)
    if kind == "b":
        return f"{t4[col]['b'][i]:>{W}.3f}"
    return f"({t4[col]['se'][i]:>{W - 2}.3f})"


print(f"{'':<16s}" + "".join(f"{c:>{W}s}" for c in COLS))
for core in ("treat", "GAP"):
    if not any(core in NAMES[c] for c in COLS):
        continue
    print(f"{core:<16s}" + "".join(_cell(c, core, "b")  for c in COLS))
    print(f"{'(SE)':<16s}" + "".join(_cell(c, core, "se") for c in COLS))
print(f"{'N':<16s}" + "".join(f"{t4[c]['n']:>{W}d}"   for c in COLS))
print(f"{'SER':<16s}" + "".join(f"{t4[c]['ser']:>{W}.3f}" for c in COLS))
```

---

## Skill 7 — Figure 1: distribution of starting wages

**用途 / Purpose.** Two stacked histograms of starting wages (February 1992 and November 1992), with $0.10 bins, New Jersey in solid black and Pennsylvania hatched. The y axis is the percentage of each state's stores, not a count. The figure is also saved to PNG for the Word export.


```python
import matplotlib.pyplot as plt

NODES = np.round(np.arange(4.25, 5.55 + 1e-9, 0.10), 2)   # 14 bins
WIDTH = 0.10


def wage_percent(wage, state):
    """Within-state percentage of stores in each $0.10 wage bin."""
    wage  = np.asarray(wage,  dtype=float)
    state = np.asarray(state, dtype=float)
    ok = np.isfinite(wage)
    idx = np.clip(np.floor((wage - NODES[0]) / WIDTH + 1e-9).astype(int),
                  0, len(NODES) - 1)
    out = {}
    for name, mask in [("NJ", ok & (state == 1.0)), ("PA", ok & (state == 0.0))]:
        n = int(mask.sum())
        cnt = np.bincount(idx[mask], minlength=len(NODES))
        out[name] = 100.0 * cnt / n if n else np.zeros(len(NODES))
        out[name + "_n"] = n
    return out


top    = wage_percent(df["WAGE_ST"],  df["STATE"])
bottom = wage_percent(df["WAGE_ST2"], df["STATE"])

fig, axes = plt.subplots(2, 1, figsize=(7.0, 10.5), dpi=130)
x = np.arange(len(NODES), dtype=float)

for ax, dat, title, ymax, ystep in [
        (axes[0], top,    "February 1992", 35, 5),
        (axes[1], bottom, "November 1992", 90, 10)]:
    ax.bar(x - 0.20, dat["NJ"], width=0.40, color="black",
           edgecolor="black", label="New Jersey", zorder=3)
    ax.bar(x + 0.20, dat["PA"], width=0.40, color="white",
           edgecolor="black", hatch="////", label="Pennsylvania", zorder=3)
    ax.set_title(title, fontsize=13)
    ax.set_xlabel("Wage Range")
    ax.set_ylabel("Percent of Stores")
    ax.set_xticks(x)
    ax.set_xticklabels([f"{v:.2f}" for v in NODES])
    ax.set_xlim(-0.65, len(NODES) - 0.35)
    ax.set_ylim(0, ymax)
    ax.set_yticks(np.arange(0, ymax + 1, ystep))
    ax.tick_params(direction="out")

handles, labels = axes[0].get_legend_handles_labels()
fig.legend(handles, labels, loc="lower center", ncol=2, frameon=False,
           bbox_to_anchor=(0.5, 0.045), fontsize=11, handlelength=2.4)
fig.suptitle("FIGURE 1.  DISTRIBUTION OF STARTING WAGE RATES", y=0.012, fontsize=11)
fig.subplots_adjust(hspace=0.32, bottom=0.13, top=0.95)
fig.savefig("figure1_starting_wages.png", dpi=200, bbox_inches="tight")
plt.show()
```

---

## Skill 8 — Independent extension: ownership heterogeneity 

**用途 / Purpose.** Two DID designs on the ownership dimension: (A) separate regressions for company-owned and franchisee stores, and (B) a pooled regression with an `own` dummy and a `treat × own` interaction. β₃ measures the extra policy effect in company-owned stores relative to franchisees.

```python
own  = (coown[m] == 1.0).astype(float)
CTL2 = np.column_stack([chain[m] == 2.0, chain[m] == 3.0,
                        chain[m] == 4.0, REG[m]]).astype(float)

NMS_A = ["const", "treat", "KFC", "Roy Rogers", "Wendy's",
         "SOUTHJ", "CENTRALJ", "PA1", "PA2"]
NMS_B = ["const", "treat", "own", "treat x own"] + NMS_A[2:]

X_A = np.column_stack([one, NJv, CTL2])
X_B = np.column_stack([one, NJv, own, NJv * own, CTL2])

sel_co, sel_fr = (own == 1.0), (own == 0.0)

rA1 = ols(Y[sel_co], X_A[sel_co], NMS_A)     # company-owned only
rA2 = ols(Y[sel_fr], X_A[sel_fr], NMS_A)     # franchisee only
rB  = ols(Y, X_B, NMS_B)                     # pooled with interaction

for name, r in [("(A1) Company-owned", rA1),
                ("(A2) Franchisee-owned", rA2),
                ("(B) Pooled + interaction", rB)]:
    print(f"\n{name}   N = {r['n']}   R2 = {r['r2']:.4f}   SER = {r['ser']:.3f}")
    print(f"{'Variable':<16s}{'Coef.':>10s}{'Std.Err.':>11s}"
          f"{'t-stat':>10s}{'p-value':>11s}")
    for i, nm in enumerate(r["names"]):
        print(f"{nm:<16s}{r['b'][i]:>10.3f}{r['se'][i]:>11.3f}"
              f"{r['t'][i]:>10.3f}{r['p'][i]:>11.4f}  {stars(r['p'][i])}")

i1 = rB["names"].index("treat")
i3 = rB["names"].index("treat x own")
print(f"\nDID effect, franchisee    : {rB['b'][i1]:.3f}")
print(f"DID effect, company-owned : {rB['b'][i1] + rB['b'][i3]:.3f}")
```

---

## Skill 9 — Export every table and figure to Word 
**用途 / Purpose.** Reads the saved `.ipynb`, pulls every text output (the printed tables) and every PNG output (Figure 1), and writes them into a Word document in cell order. Save the notebook first — the `.ipynb` holds the full output even when the notebook display truncates it.


```python
# requires:  %pip install python-docx
import io, json, base64, os
from docx import Document
from docx.shared import Pt, Inches
from docx.enum.text import WD_ALIGN_PARAGRAPH

NB_PATH  = "Table复刻过程.ipynb"        # <-- set to your notebook's file name
OUT_DOCX = "replication_report.docx"

with open(NB_PATH, "r", encoding="utf-8") as fh:
    nb = json.load(fh)


def as_text(v):
    return "".join(v) if isinstance(v, list) else str(v)


doc = Document()
sec = doc.sections[0]
sec.left_margin = sec.right_margin = Inches(0.7)
doc.add_heading("Card & Krueger (1994): Replication Output", level=0)

n_txt = n_img = 0
for i, cell in enumerate(nb.get("cells", [])):
    if cell.get("cell_type") != "code":
        continue
    src = as_text(cell.get("source", [])).strip()
    texts, pngs = [], []
    for out in cell.get("outputs", []):
        data = out.get("data", {})
        if out.get("output_type") == "stream":
            texts.append(as_text(out.get("text", "")))
        if "text/plain" in data:
            texts.append(as_text(data["text/plain"]))
        if "image/png" in data:
            pngs.append(as_text(data["image/png"]))
    if not texts and not pngs:
        continue

    doc.add_heading(src.splitlines()[0][:80] if src else f"Cell {i}", level=2)
    for t in texts:
        if t.strip():
            run = doc.add_paragraph().add_run(t.rstrip())
            run.font.name = "Consolas"
            run.font.size = Pt(8)
            n_txt += 1
    for b in pngs:
        doc.add_picture(io.BytesIO(base64.b64decode(b)), width=Inches(6.4))
        doc.paragraphs[-1].alignment = WD_ALIGN_PARAGRAPH.CENTER
        n_img += 1
    doc.add_paragraph()

doc.save(OUT_DOCX)
print(f"OK -> {os.path.abspath(OUT_DOCX)}  ({n_txt} text blocks, {n_img} figures)")
```

---

## Skill 10 — Export the analysis dataset to Excel 

**用途 / Purpose.** Writes a copy of the data with the derived columns (FTE1, FTE2, DELTA_FTE) and the Table 4 sample flag. `df` itself is never modified. Falls back to CSV if `openpyxl` is unavailable.


```python
# requires:  %pip install openpyxl     (xlsx version only)
out = df.copy()

out["FTE1"]      = fte1
out["FTE2"]      = fte2
out["DELTA_FTE"] = delta_fte
out["IN_SAMPLE"] = m

OUT_XLSX = r"C:\Users\lin\Desktop\6067B\PART 2\个人项目\public_analysis.xlsx"

with pd.ExcelWriter(OUT_XLSX, engine="openpyxl") as xw:
    out.to_excel(xw, index=False, sheet_name="full")
    out[out["IN_SAMPLE"]].to_excel(xw, index=False, sheet_name="sample")

print("full     :", out.shape)
print("in-sample:", int(out["IN_SAMPLE"].sum()))     # should print 351
```

```python
# ---- CSV fallback, no dependency -----------------------------------------
out.to_csv("public_analysis_full.csv", index=False, encoding="utf-8-sig")
out[out["IN_SAMPLE"]].to_csv("public_analysis_sample.csv",
                             index=False, encoding="utf-8-sig")
```

---

## Notes on conventions 

- **Means** use only the non-missing responses for that variable; there is no listwise deletion. 
- **Standard errors** are `s / √n`; the parentheses in Tables 2 and 3 contain standard errors, not standard deviations. 
- **Percentages** are printed as raw numbers (41.1, not 0.411). 
- **Table 3 row 3** differences two overlapping means, so its variance carries the covariance term with `RHO_W1W2 = 0.45`. 
- **Table 4 sample** requires employment and starting wages in both waves; with the public-use extract this yields 351 stores against the paper's 357. 
