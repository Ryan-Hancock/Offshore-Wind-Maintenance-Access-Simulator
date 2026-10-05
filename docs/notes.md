# Failure Definition
There is no standardized way for defining a failure in the wind energy industry. This analysis defines a
failure as a visit to a turbine, outside of a scheduled operation, in which material is consumed. 

source: Failure Rate, Repair Time and Unscheduled O&M Cost
Analysis of Offshore Wind Turbines - page 5

# Failure Rate Calulcation

The formula is a normalised average: total failures, divided by how many turbines there were and how many years you watched them.

$$\lambda = \frac{\sum_{i=1}^{I}\sum_{k=1}^{K} n_{i,k}/N_i}{\sum_{i=1}^{I} T_i/8760}$$

**Numerator: failures per turbine.** The data comes in intervals (i), such as reporting periods. For each interval, you add up the failures across every subassembly (k: gearbox, generator, pitch system and so on), then divide by the number of turbines in that interval, N_i. Dividing inside the sum matters because the population can change between intervals as farms are added. Summing over the intervals gives the failures one average turbine had over the whole observation period.

**Denominator: years observed.** T_i is the length of each interval in hours, and 8760 is the number of hours in a year (24 × 365), so this is the total observation time in years.

**Result:** failures per turbine, divided by years, gives failures per turbine per year.

## Worked example

| Interval | Hours (T_i) | Turbines (N_i) | Failures across subassemblies | Per turbine |
|---|---|---|---|---|
| 1 | 8760 | 100 | 30 + 50 + 120 = 200 | 2.0 |
| 2 | 4380 | 120 | 20 + 28 + 72 = 120 | 1.0 |

- Numerator: 2.0 + 1.0 = 3.0 failures per turbine
- Denominator: (8760 + 4380) / 8760 = 1.5 years
- λ = 3.0 / 1.5 = **2.0 failures per turbine per year**

## Code

```python
HOURS_PER_YEAR = 8760

intervals = [
    {"hours": 8760, "turbines": 100,
     "failures": {"gearbox": 30, "generator": 50, "pitch": 120}},
    {"hours": 4380, "turbines": 120,
     "failures": {"gearbox": 20, "generator": 28, "pitch": 72}},
]

def failure_rate(intervals, subassembly=None):
    """Failures per turbine per year. Pass a subassembly name for its rate alone."""
    failures_per_turbine = 0.0
    for iv in intervals:
        if subassembly:
            n = iv["failures"][subassembly]      # one k only
        else:
            n = sum(iv["failures"].values())     # sum over k
        failures_per_turbine += n / iv["turbines"]   # sum over i of n_ik / N_i

    years = sum(iv["hours"] for iv in intervals) / HOURS_PER_YEAR
    return failures_per_turbine / years

print(failure_rate(intervals))             # 2.0
print(failure_rate(intervals, "gearbox"))  # ~0.311
```

- **Subassembly rates add up.** Dropping the sum over k gives the rate for one subassembly, and those rates sum to the overall λ. That is how the paper breaks its total into per-component figures.
- **It is a flat average.** The formula assumes a constant failure rate over the period, so it hides any early-life or wear-out trends.

The cost categories are a separate labelling step: each failure is binned by material cost alone as a minor repair (under €1,000), a major repair (€1,000 to €10,000) or a major replacement (over €10,000). You can then compute a λ for each bin the same way, by counting only the failures in that bin.

source: Failure Rate, Repair Time and Unscheduled O&M Cost
Analysis of Offshore Wind Turbines - page 6

# Cleaned Up Data

'SQL' data was also cleaned to remove any scheduled operations such as scheduled services or scheduled
inspections. These scheduled events may influence the failure rates as poorly maintained turbines may have
higher failure rates.

source: Failure Rate, Repair Time and Unscheduled O&M Cost
Analysis of Offshore Wind Turbines - page 6

# Failure Workflow

This is a data pipeline that turns raw maintenance records into four numbers per subassembly and cost category: failure rate, average material cost, average repair time and average number of technicians.

## The workflow

1. **Get the data.** Work orders (one per repair job) and material usage records (the parts consumed by each job) come from the manufacturer.
2. **Clean it.** SQL and Excel are used to join the two sources and remove junk such as duplicates and non-failure work orders.
3. **Cost each failure.** Add up the material costs on each work order.
4. **Categorise by cost.** Under €1,000 is a minor repair, €1,000 to €10,000 a major repair, over €10,000 a major replacement.
5. **Assign a subassembly.** Someone reads the free-text description on each work order and decides whether it was the gearbox, generator, pitch system and so on. This is manual, and the most labour-intensive step.
6. **Average.** Compute the mean repair time and mean technician count for each group.

## Equations

The paper gives no equations for this part, so these are my formalisation of the flow chart. For a work order w that uses materials m = 1…M_w:

**Cost of one failure (step 3)**

$$C_w = \sum_{m=1}^{M_w} c_{w,m}$$

**Category (step 4)**

$$\text{cat}(w) = \begin{cases} \text{minor repair} & C_w < 1000 \\ \text{major repair} & 1000 \le C_w \le 10000 \\ \text{major replacement} & C_w > 10000 \end{cases}$$

**Averages (step 6 and the cost average in the text).** Let W be the set of work orders for subassembly k in a given category:

$$\bar{C} = \frac{1}{|W|}\sum_{w \in W} C_w \qquad \bar{t} = \frac{1}{|W|}\sum_{w \in W} t_w \qquad \bar{p} = \frac{1}{|W|}\sum_{w \in W} p_w$$

where t_w is the repair time in hours and p_w is the number of technicians on the job.

The count |W| is also the n that feeds the failure rate formula from the previous question, which is how the two sections connect.

## Code

```python
import pandas as pd

# Step 1-2: cleaned inputs
work_orders = pd.DataFrame({
    "wo_id":       [1, 2, 3, 4, 5],
    "subassembly": ["gearbox", "gearbox", "pitch", "pitch", "generator"],  # step 5, assigned by hand
    "repair_hours": [8, 52, 4, 6, 30],
    "technicians":  [2, 5, 2, 2, 4],
})

materials = pd.DataFrame({
    "wo_id": [1, 1, 2, 3, 4, 4, 5],
    "cost":  [300, 250, 85000, 120, 900, 2400, 6500],
})

# Step 3: cost of each failure = sum of its material costs
cost = materials.groupby("wo_id")["cost"].sum().rename("material_cost")
df = work_orders.join(cost, on="wo_id")

# Step 4: categorise by cost
df["category"] = pd.cut(
    df["material_cost"],
    bins=[0, 1000, 10000, float("inf")],
    labels=["minor repair", "major repair", "major replacement"],
    right=False,
)

# Step 6: averages per subassembly and category
summary = (
    df.groupby(["subassembly", "category"], observed=True)
      .agg(failures=("wo_id", "count"),
           avg_cost=("material_cost", "mean"),
           avg_repair_hours=("repair_hours", "mean"),
           avg_technicians=("technicians", "mean"))
)
print(summary)
```

Output:

```
                               failures  avg_cost  avg_repair_hours  avg_technicians
subassembly category
gearbox     minor repair              1     550.0               8.0              2.0
            major replacement         1   85000.0              52.0              5.0
generator   major repair              1    6500.0              30.0              4.0
pitch       minor repair              1     120.0               4.0              2.0
            major repair              1    3300.0               6.0              2.0
```

## Caveats

- **Boundary handling is my assumption.** The paper says "between €1,000 and €10,000" without saying which bin exactly €1,000 or €10,000 falls into. The code puts €1,000 in major repair and €10,000 in major replacement, which differs slightly from the equation above at the upper edge. It will rarely matter in practice.
- **Step 5 is not reproducible in code as written.** It depends on a person's reading of free text, so two analysts could classify the same work order differently.
- **Costs are materials only.** Labour, vessel hire and downtime are excluded, which is why the repair time and technician averages are reported separately.

# Failure Workflow Probability

**Poisson process**: they arrive randomly at a constant average rate λ, independently of each other.

## The core equations

**Number of failures in time t (years)**

$$P(N = n) = \frac{(\lambda t)^n e^{-\lambda t}}{n!}$$

**Probability of at least one failure**

$$P(N \ge 1) = 1 - e^{-\lambda t}$$

**Reliability (probability of surviving to time t with no failure)**

$$R(t) = e^{-\lambda t}$$

**Mean time between failures**

$$\text{MTBF} = 1/\lambda$$

**Farm level.** For M turbines, rates add, so the farm sees failures at rate Mλ.

**Given a failure, how bad is it?** Using the category rates from the workflow:

$$P(\text{category } c \mid \text{failure of subassembly } k) = \frac{\lambda_{k,c}}{\sum_{c'} \lambda_{k,c'}}$$

**Expected annual material cost per turbine**, which joins the two sections you've read:

$$E[\text{cost}] = \sum_{k}\sum_{c} \lambda_{k,c}\,\bar{C}_{k,c}$$

## With the earlier numbers

Using λ = 2.0 overall and λ ≈ 0.311 for the gearbox:

| Question | Calculation | Answer |
|---|---|---|
| A turbine has no failures in a year | e^(−2.0) | 13.5% |
| A turbine has exactly 2 failures in a year | 2² e^(−2) / 2! | 27.1% |
| At least one gearbox failure in a year | 1 − e^(−0.311) | 26.7% |
| Mean time between failures (any) | 1 / 2.0 | 0.5 years |
| Expected gearbox failures, 100-turbine farm | 100 × 0.311 | 31 per year |

## Code

```python
import math
import numpy as np

lam_total = 2.0
lam_gearbox = 0.4667 / 1.5   # ~0.311 from the earlier example

def p_exactly(n, lam, t=1.0):
    return (lam * t) ** n * math.exp(-lam * t) / math.factorial(n)

def p_at_least_one(lam, t=1.0):
    return 1 - math.exp(-lam * t)

print(p_exactly(0, lam_total))          # 0.135
print(p_exactly(2, lam_total))          # 0.271
print(p_at_least_one(lam_gearbox))      # 0.267
print(1 / lam_total)                    # MTBF: 0.5 years

# Expected annual material cost per turbine: sum of rate x average cost
rates_and_costs = [
    # (subassembly, category, failures/turbine/year, avg material cost)
    ("gearbox", "minor repair",      0.25,  550),
    ("gearbox", "major replacement", 0.06,  85000),
    ("pitch",   "minor repair",      0.80,  120),
    ("pitch",   "major repair",      0.15,  3300),
]
expected_cost = sum(lam * cost for _, _, lam, cost in rates_and_costs)
print(expected_cost)                    # 5828.5 per turbine per year

# Monte Carlo: distribution of annual cost for a 100-turbine farm
rng = np.random.default_rng(42)
turbines, runs = 100, 10_000
annual = np.zeros(runs)
for _, _, lam, cost in rates_and_costs:
    annual += rng.poisson(lam * turbines, size=runs) * cost

print(annual.mean())                    # close to 582,850
print(np.percentile(annual, [5, 95]))   # the spread a planner would budget for
```

The rates and costs in that table are made up for illustration; the paper's own tables give the real ones. The simulation is the useful part in practice, because the mean hides how much the rare, expensive replacements swing the total from year to year.

## What the model assumes

- **Constant rate.** Real turbines tend to fail more when new and again when old (the "bathtub curve"). A Weibull distribution handles that, but needs failure times, not just a rate.
- **Independence.** One failure doesn't trigger another, and turbines don't fail together. Storms and shared faults break this.
- **λ is itself an estimate.** It came from a finite sample, so it has its own uncertainty, which matters most for rare events like major replacements.
- **Fixed cost per category.** The simulation uses the average cost each time; real costs vary within a category.

# Labour Averages

This section gives the crew size per repair, which is the last ingredient needed to cost labour: you now have how often things fail, how long repairs take and how many people they need.

## Summary

- **What is measured:** for each subassembly and failure category, the average number of technicians who logged any time against a repair.
- **Why it matters:** combined with failure rate and repair time, it gives labour cost in an O&M model.
- **Main finding:** minor and major repairs need about 2 to 3 technicians almost regardless of subassembly. Major replacements are the outliers: blades 21.0, gearbox 17.2, hub 10.0, generator 7.9.
- **Gearbox dominates labour cost.** Blades need more people per replacement, but gearboxes fail far more often, so they contribute more over a year.
- **The headcount overstates effort.** Twenty technicians on a job doesn't mean twenty people for the full duration. More likely a small core team works throughout and others log short supporting stints.
- **Reading the table:** a 0.0 means no failures of that type were recorded (for example, no sensor major replacements), not that zero technicians were needed.

## Equations

The paper gives none here, so these are my formalisation.

**Average technicians**, where W is the set of work orders for subassembly k in category c, and p_w is the number of distinct technicians who logged time on work order w:

$$\bar{p}_{k,c} = \frac{1}{|W_{k,c}|}\sum_{w \in W_{k,c}} p_w$$

**Annual labour cost per turbine**, with repair time t̄ in hours and an hourly wage r:

$$L = r \sum_{k}\sum_{c} \lambda_{k,c}\;\bar{p}_{k,c}\;\bar{t}_{k,c}$$

This is an upper bound, because it assumes every technician works the full repair time. If you had the raw time logs, the accurate version replaces p̄ × t̄ with the average technician-hours actually booked:

$$L = r \sum_{k}\sum_{c} \lambda_{k,c}\;\bar{h}_{k,c}$$

## Code

Technician counts are from Figure 15. The failure rates, repair times and wage are placeholders, so swap in the paper's own tables.

```python
import pandas as pd

WAGE = 50  # per technician-hour, placeholder

df = pd.DataFrame(
    [
        # subassembly, category, technicians (Fig. 15), failures/turbine/yr, repair hours
        ("gearbox", "minor repair",      2.2,  0.40,    8),
        ("gearbox", "major repair",      3.2,  0.04,   22),
        ("gearbox", "major replacement", 17.2, 0.15,  231),
        ("blades",  "minor repair",      2.1,  0.45,    9),
        ("blades",  "major repair",      3.3,  0.01,   21),
        ("blades",  "major replacement", 21.0, 0.001, 288),
    ],
    columns=["subassembly", "category", "technicians", "rate", "repair_hours"],
)

# Technician-hours per turbine per year = rate x crew size x repair time
df["tech_hours"] = df["rate"] * df["technicians"] * df["repair_hours"]
df["labour_cost"] = df["tech_hours"] * WAGE

print(df[["subassembly", "category", "tech_hours", "labour_cost"]])
print(df.groupby("subassembly")[["tech_hours", "labour_cost"]].sum())
```

Output:

```
  subassembly           category  tech_hours  labour_cost
0     gearbox       minor repair       7.040       352.00
1     gearbox       major repair       2.816       140.80
2     gearbox  major replacement     595.980     29799.00
3      blades       minor repair       8.505       425.25
4      blades       major repair       0.693        34.65
5      blades  major replacement       6.048       302.40

             tech_hours  labour_cost
subassembly
blades           15.246        762.3
gearbox         605.836      30291.8
```

Even with placeholder rates, this reproduces the paper's point: blades need a bigger crew per replacement, but the gearbox's higher failure rate makes it the larger labour cost. The gearbox replacement figure also shows the overstatement problem, since it assumes all 17 technicians work all 231 hours.

When loading the full Figure 15 table, treat the 0.0 entries as missing values, not zeros, or they will drag down any averages you take across subassemblies.

# Wave Height States

This table converts significant wave height into a 0–9 sea state code, which matters for O&M because sea state decides whether technicians can reach a turbine at all.

## Summary

- **Hs is significant wave height:** the average height of the highest third of waves, in metres. It roughly matches what an experienced observer would estimate by eye.
- **The code is a binning of Hs.** Each code covers a range, and the ranges widen as conditions get worse.
- **Link to the earlier sections:** repair time and technician counts only apply once a crew is on the turbine. Vessels have an Hs limit for safe transfer (around 1.5 m is a common figure for crew transfer vessels), so codes 0–3 are mostly workable, code 4 is marginal and anything above means waiting. That waiting time is the "distance from shore and weather" cost the Carroll paper deliberately left out.
- **Naming issue:** these ranges are the Douglas sea scale (the WMO sea state code), not the Beaufort scale. Beaufort measures wind force and runs 0–12. The source has mislabelled it, which is a common mix-up, so cite it as Douglas or WMO sea state in your notes.

## Equations

**Significant wave height** from a record of sea surface elevation η, where σ is its standard deviation and m₀ is the variance (the zeroth moment of the wave spectrum):

$$H_s = 4\sigma_\eta = 4\sqrt{m_0}$$

The older definition, from counting individual waves, is the mean of the highest third:

$$H_{1/3} = \frac{1}{N/3}\sum_{j=1}^{N/3} H_j \quad \text{(waves sorted highest first)}$$

**Sea state code** is a step function of Hs:

$$S(H_s) = \begin{cases} 0 & H_s = 0 \\ 1 & 0 < H_s \le 0.1 \\ 2 & 0.1 < H_s \le 0.5 \\ 3 & 0.5 < H_s \le 1.25 \\ 4 & 1.25 < H_s \le 2.5 \\ 5 & 2.5 < H_s \le 4 \\ 6 & 4 < H_s \le 6 \\ 7 & 6 < H_s \le 9 \\ 8 & 9 < H_s \le 14 \\ 9 & H_s > 14 \end{cases}$$

**Accessibility**, the fraction of time a vessel with limit H_lim can work:

$$A = P(H_s \le H_{lim}) \approx \frac{\text{hours with } H_s \le H_{lim}}{\text{total hours}}$$

## Code

```python
import numpy as np

BINS = [0.1, 0.5, 1.25, 2.5, 4.0, 6.0, 9.0, 14.0]
NAMES = ["Calm (glassy)", "Calm (rippled)", "Smooth", "Slight", "Moderate",
         "Rough", "Very rough", "High", "Very high", "Huge"]

def sea_state(hs):
    """Sea state code 0-9 from significant wave height in metres."""
    hs = np.asarray(hs, dtype=float)
    return np.where(hs == 0, 0, np.digitize(hs, BINS, right=True) + 1)

def significant_wave_height(elevation):
    """Hs from a time series of sea surface elevation in metres."""
    return 4 * np.std(elevation)

def accessibility(hs, limit=1.5):
    """Fraction of records where Hs is within the vessel limit."""
    return np.mean(np.asarray(hs) <= limit)

hs = [0, 0.05, 0.3, 1.0, 1.4, 2.0, 3.1, 5.0, 7.5, 15]

for h, s in zip(hs, sea_state(hs)):
    print(f"{h:>5} m -> {s} {NAMES[s]}")

print(accessibility(hs))   # 0.5
```

Output:

```
    0 m -> 0 Calm (glassy)
 0.05 m -> 1 Calm (rippled)
  0.3 m -> 2 Smooth
  1.0 m -> 3 Slight
  1.4 m -> 4 Moderate
  2.0 m -> 4 Moderate
  3.1 m -> 5 Rough
  5.0 m -> 6 Very rough
  7.5 m -> 7 High
   15 m -> 9 Huge
0.5
```

source: Innovative Hull Design for Servicing Offshore Wind TurbinesCORA - page: 62

# Wave Temp

This section describes the wave climate in two parts: a scatter diagram saying how often each sea condition occurs, and the JONSWAP spectrum saying how wave energy is spread across frequencies within one condition.

## Summary

- **Scatter diagram:** each cell counts how many observations fell in that combination of significant wave height (Hs, rows) and peak period (Tp, columns). It is a 2D histogram of the sea.
- **Tp is peak period:** the period of the most energetic waves, in seconds. Vessel motion depends on it as much as on height, since a 2 m sea at 6 s is steep and choppy while 2 m at 14 s is a gentle swell.
- **What this one shows:** the most common condition is Hs 1–2 m with Tp around 7–8 s (the 6,292 cell), and most observations sit between 5.5 and 12.5 s. The blank lower-left is physical: tall waves can't have short periods because they would break.
- **JONSWAP:** a standard formula for the wave spectrum in fetch-limited seas like the North Sea. It is more sharply peaked than the open-ocean (Pierson-Moskowitz) spectrum, which suits the Atlantic conditions the thesis mentions.
- **Link to earlier notes:** summing rows gives the probability of each Hs band, which is the accessibility figure from the sea state section.

## Equations

**Joint probability** of a sea condition, from cell counts n:

$$p_{ij} = \frac{n_{ij}}{\sum_i\sum_j n_{ij}}$$

**Probability Hs is below a limit** (sum whole rows, across all periods):

$$P(H_s \le h) = \sum_{i:\,H_{s,i} \le h}\;\sum_j p_{ij}$$

**JONSWAP spectrum**, with peak frequency f_p = 1/Tp:

$$S(f) = \frac{\alpha g^2}{(2\pi)^4 f^5}\,\exp\!\left[-\frac{5}{4}\left(\frac{f_p}{f}\right)^4\right]\gamma^{\,\exp\left[-\frac{(f-f_p)^2}{2\sigma^2 f_p^2}\right]}$$

- γ is the peak enhancement factor, typically 3.3 (γ = 1 gives Pierson-Moskowitz)
- σ = 0.07 for f ≤ f_p and 0.09 for f > f_p
- α sets the overall energy level; in practice you scale it so the spectrum matches the Hs you want

**Tie back to Hs**, using the area under the spectrum:

$$m_0 = \int_0^\infty S(f)\,df \qquad H_s = 4\sqrt{m_0}$$

## Code

```python
import numpy as np

TP = np.arange(2.5, 21, 1.0)    # 2.5 ... 20.5 s
HS = np.arange(0.5, 15, 1.0)    # 0.5 ... 14.5 m (class midpoints)

# (index of first Tp column, counts) for each Hs row, read from Figure 2.24
ROWS = [
    (0,  [42, 429, 1172, 1662, 1634, 1291, 888, 558, 331, 189, 105, 57, 31, 17, 9]),
    (0,  [2, 109, 964, 3034, 5260, 6292, 5902, 4689, 3317, 2160, 1325, 778, 443, 246, 135, 73, 39, 21, 11]),
    (1,  [2, 76, 578, 1837, 3373, 4297, 4248, 3504, 2532, 1659, 1009, 581, 320, 171, 89, 46, 23, 11]),
    (2,  [2, 47, 320, 1006, 1881, 2442, 2436, 2000, 1419, 900, 523, 285, 147, 73, 35, 16, 7]),
    (3,  [2, 33, 206, 635, 1181, 1519, 1485, 1179, 797, 475, 257, 128, 60, 27, 12, 5]),
    (4,  [1, 22, 133, 395, 715, 888, 829, 619, 388, 212, 103, 46, 19, 7, 3]),
    (5,  [1, 16, 85, 242, 414, 485, 420, 287, 163, 79, 34, 13, 5, 2]),
    (6,  [1, 11, 55, 144, 228, 244, 191, 117, 58, 25, 9, 3, 1]),
    (7,  [1, 8, 35, 83, 117, 112, 77, 41, 18, 6, 2, 1]),
    (8,  [1, 6, 22, 45, 55, 45, 27, 12, 4, 1]),
    (9,  [1, 4, 13, 22, 23, 16, 8, 3, 1]),
    (10, [1, 3, 7, 10, 9, 5, 2, 1]),
    (12, [2, 4, 4, 3, 1]),
    (13, [1, 2, 1, 1]),
    (15, [1]),
]

counts = np.zeros((len(HS), len(TP)))
for i, (start, values) in enumerate(ROWS):
    counts[i, start:start + len(values)] = values

p = counts / counts.sum()               # joint probability
p_hs = p.sum(axis=1)                    # marginal over Tp

i, j = np.unravel_index(p.argmax(), p.shape)
print(f"Most common: Hs {HS[i]} m, Tp {TP[j]} s ({p[i, j]:.1%})")
print(f"P(Hs < 1 m) = {p_hs[:1].sum():.1%}")
print(f"P(Hs < 2 m) = {p_hs[:2].sum():.1%}")


def jonswap(f, hs, tp, gamma=3.3):
    """JONSWAP spectrum S(f) in m^2/Hz, scaled to the requested Hs."""
    fp = 1 / tp
    sigma = np.where(f <= fp, 0.07, 0.09)
    peak = gamma ** np.exp(-((f - fp) ** 2) / (2 * sigma**2 * fp**2))
    shape = f**-5 * np.exp(-1.25 * (fp / f) ** 4) * peak
    m0 = np.sum(shape) * (f[1] - f[0])
    return shape * (hs**2 / 16) / m0    # scale so that 4*sqrt(m0) = hs

f = np.linspace(0.02, 1.0, 2000)
S = jonswap(f, hs=1.5, tp=7.5)
print(f"Check Hs = {4 * np.sqrt(np.sum(S) * (f[1] - f[0])):.2f} m")
```

Expected output:

```
Most common: Hs 1.5 m, Tp 7.5 s (6.4%)
P(Hs < 1 m) = 8.6%
P(Hs < 2 m) = 44.2%
Check Hs = 1.50 m
```

So at this site a vessel limited to under 2 m could work a bit under half the time, and one limited to under 1 m less than a tenth, which is the thesis's argument for better access vessels.


source: Innovative Hull Design for Servicing Offshore Wind TurbinesCORA - page: 62

# Vessel Accesibility

This figure answers the question the last two sections were building to: for a vessel that can work up to a given wave height, what share of the year can it reach the turbines at each site?

## Summary

- **What is plotted:** year-round accessibility (share of time the sea is within the limit) against four vessel Hs limits, for five sites.
- **Approximate values read from the chart:**

| Site | 1 m | 1.5 m | 2 m | 2.5 m |
|---|---|---|---|---|
| North Sea, 37 km | 45% | 68% | 83% | 99% |
| North Sea, 100 km | 36% | 60% | 76% | 87% |
| M2, Irish Sea | 47% | 71% | 86% | 94% |
| M1, 71 km | 2% | 13% | 27% | 45% |
| M3, 36 km | 3% | 16% | 33% | 49% |

- **Vessel capability pays off most at the low end.** At the North Sea 37 km site, going from a 1 m to a 1.5 m limit adds about 23 points of access; going from 2 m to 2.5 m adds about 16.
- **Exposure matters more than distance.** M3 is only 36 km out but is almost inaccessible at 1 m, while the North Sea site at 100 km manages 36%. M1 and M3 look like the Irish weather buoys on the Atlantic side, which fits the thesis's point about harsher, longer-period Atlantic conditions; that identification is my inference, not stated in the figure.
- **Sheltered sites are the benchmark.** The Irish Sea buoy (M2) is as good as or better than the near-shore North Sea site until the 2.5 m limit.
- **Cross-check with the scatter diagram:** that table gave about 44% of time under 2 m, well below the North Sea sites here (76–83%). It evidently describes a more exposed location, so don't mix the two datasets in one calculation.

## Equations

**Simple accessibility** from N records of Hs:

$$A(H_{lim}) = \frac{1}{N}\sum_{t=1}^{N} \mathbb{1}\left[H_{s,t} \le H_{lim}\right]$$

**Weather window accessibility**, the stricter version: a job needing τ hours can only start at time t if the sea stays within the limit for the whole window:

$$A_\tau(H_{lim}) = \frac{1}{N}\sum_{t=1}^{N} \mathbb{1}\left[\max_{t \le s < t+\tau} H_{s,s} \le H_{lim}\right]$$

A_τ is always less than or equal to A, and the gap widens as τ grows. The figure doesn't say which definition or window length it uses, so check reference [95] before quoting the numbers.

**Downtime per failure**, which is where this joins the Carroll failure data:

$$D = t_{wait} + t_{travel} + t_{repair}$$

**Annual downtime per turbine** (hours), summing over subassemblies and categories:

$$D_{year} = \sum_k\sum_c \lambda_{k,c}\left(\bar{t}_{wait} + t_{travel} + \bar{t}_{k,c}\right)$$

Low accessibility hurts through t_wait: a short repair can cost weeks of lost generation if the failure happens at the start of a long rough spell.

## Code

There is no raw data behind the figure, so this generates a synthetic hourly Hs series to show the calculation. Replace it with real buoy or hindcast data.

```python
import numpy as np

def accessibility(hs, limit):
    """Share of time Hs is within the limit."""
    return np.mean(hs <= limit)

def window_starts(hs, limit, window_hours):
    """Indices where a full weather window of the given length begins."""
    ok = (hs <= limit).astype(int)
    run = np.convolve(ok, np.ones(window_hours, dtype=int), mode="valid")
    return np.flatnonzero(run == window_hours)

def window_accessibility(hs, limit, window_hours):
    """Share of start times with a full workable window ahead."""
    return len(window_starts(hs, limit, window_hours)) / len(hs)

def mean_wait(hs, limit, window_hours):
    """Average hours from a random failure time to the next workable window."""
    starts = window_starts(hs, limit, window_hours)
    t = np.arange(starts[-1] + 1)                 # only times with a window ahead
    next_start = starts[np.searchsorted(starts, t)]
    return np.mean(next_start - t)

# Synthetic hourly Hs for 10 years: lognormal with hour-to-hour persistence
rng = np.random.default_rng(1)
n, phi, mu, sd = 10 * 8760, 0.98, np.log(1.3), 0.6
x = np.zeros(n)
for i in range(1, n):
    x[i] = phi * x[i - 1] + rng.normal(0, sd * np.sqrt(1 - phi**2))
hs = np.exp(mu + x)

print("limit  simple  8h-window  mean wait (h)")
for limit in [1.0, 1.5, 2.0, 2.5]:
    print(f"{limit:>4} m  {accessibility(hs, limit):6.1%}  "
          f"{window_accessibility(hs, limit, 8):9.1%}  "
          f"{mean_wait(hs, limit, 8):8.1f}")
```

I haven't run this, so I've left out the output; expect the 8-hour window column to sit below the simple column at every limit, and the mean wait to fall steeply as the limit rises.

## Caveats

- **Chart values are eyeballed** to within a couple of points.
- **Year-round averages hide seasons.** Winter accessibility is far lower than summer, and failures don't wait for summer, so O&M models usually work month by month.
- **Hs alone is a simplification.** Real transfer limits also depend on wave period, direction, wind and daylight, which is why the previous section tracked Tp as well.
- **The synthetic series is a toy.** Its persistence and distribution are invented, so its numbers mean nothing for a real site; only the method carries over.

source: Innovative Hull Design for Servicing Offshore Wind TurbinesCORA - page: 65

# Crew Limit

The passage gives three different answers depending on what the crew are doing, so I've separated them. All values are RMS (root mean square) of the vessel's motion, with accelerations as a fraction of g.

## The limits

**Personnel on a wind farm service vessel (WFSV)**

| Situation | Vertical accel. | Lateral / horizontal accel. | Roll | Other | Source |
|---|---|---|---|---|---|
| Step-across transfer to turbine | 0.05g (in saloon) | 0.04g | 3° | | [88] |
| Transit, comfort criterion (passengers) | 0.05g | 0.04g | 2.5° | | [97] |
| Transit, reported in WFSV practice | 0.15g | 0.12g | 6° | Pitch 4°, 5–10 slams per hour | [88] |

**General criteria by type of work, Table 2.2 [97]**

| Activity | Vertical accel. | Lateral accel. | Roll |
|---|---|---|---|
| Light manual work | 0.20g | 0.10g | 6.0° |
| Heavy manual work | 0.15g | 0.07g | 4.0° |
| Intellectual work | 0.10g | 0.05g | 3.0° |
| Transit passengers | 0.05g | 0.04g | 2.5° |
| Cruise liner | 0.02g | 0.03g | 2.0° |

**Vessel operability, fast small craft (the boat's limit, not the people's)**

| Quantity | Limit |
|---|---|
| Vertical accel. at forward perpendicular | 0.275g |
| Vertical accel. at bridge | 0.2g |
| Lateral accel. at bridge | 0.1g |
| Roll | 4.0° |
| Slamming probability | 0.03 |
| Deck wetness probability | 0.05 |

## Which one to use

- **For "can technicians transfer":** 0.05g vertical, 0.04g horizontal, 3° roll. This is the binding limit for access.
- **For "will technicians arrive fit to work":** the two transit rows disagree by a factor of three. The 0.05g passenger figure is a comfort standard; 0.15g is what WFSV operators reportedly accept. The gap is the seasickness problem: crews tolerate the higher motion, but it is closer to the "heavy manual work" level than to passenger comfort.
- **Table 2.2 is not wind-specific.** It is a general seakeeping standard, and the thesis itself says these limits suit moving around the deck more than the transfer itself.

## Equations

**RMS of a motion record** x (acceleration or angle), measured about its mean:

$$x_{RMS} = \sqrt{\frac{1}{N}\sum_{t=1}^{N}\left(x_t - \bar{x}\right)^2}$$

**Acceleration in g:** divide by 9.81 m/s².

**Operable** only if every motion is within its limit:

$$\text{OK} = \bigwedge_{m}\left[x_{RMS,m} \le L_m\right]$$

**Link to the wave notes.** RMS motion is the square root of the area under the vessel's response spectrum, which is the wave spectrum filtered by the vessel's response (its RAO, the motion per metre of wave at each frequency):

$$x_{RMS} = \sqrt{\int_0^\infty \left|RAO(f)\right|^2 S(f)\,df}$$

This is what turns these motion limits into a limiting Hs for each Tp, and so into the accessibility percentages in the weather window figure.

## Code

```python
import numpy as np

G = 9.81

# RMS limits: accelerations in g, angles in degrees
LIMITS = {
    "transfer (step-across)":   {"vertical": 0.05, "lateral": 0.04, "roll": 3.0},
    "transit, comfort":         {"vertical": 0.05, "lateral": 0.04, "roll": 2.5},
    "transit, WFSV practice":   {"vertical": 0.15, "lateral": 0.12, "roll": 6.0, "pitch": 4.0},
    "light manual work":        {"vertical": 0.20, "lateral": 0.10, "roll": 6.0},
    "heavy manual work":        {"vertical": 0.15, "lateral": 0.07, "roll": 4.0},
    "intellectual work":        {"vertical": 0.10, "lateral": 0.05, "roll": 3.0},
}

def rms(x):
    x = np.asarray(x, dtype=float)
    return np.sqrt(np.mean((x - x.mean()) ** 2))

def check(measured, criterion):
    """Return (ok, list of exceeded motions) for one criterion."""
    exceeded = [m for m, limit in LIMITS[criterion].items()
                if m in measured and measured[m] > limit]
    return not exceeded, exceeded

# Example motion record: 10 minutes at 10 Hz
t = np.arange(0, 600, 0.1)
vertical_ms2 = 1.0 * np.sin(2 * np.pi * t / 6)    # m/s^2
lateral_ms2  = 0.4 * np.sin(2 * np.pi * t / 8)    # m/s^2
roll_deg     = 4.0 * np.sin(2 * np.pi * t / 10)   # degrees

measured = {
    "vertical": rms(vertical_ms2) / G,
    "lateral":  rms(lateral_ms2) / G,
    "roll":     rms(roll_deg),
}
print({k: round(v, 3) for k, v in measured.items()})

for name in LIMITS:
    ok, exceeded = check(measured, name)
    print(f"{name:<26} {'OK' if ok else 'EXCEEDED: ' + ', '.join(exceeded)}")
```

Expected output:

```
{'vertical': 0.072, 'lateral': 0.029, 'roll': 2.828}
transfer (step-across)     EXCEEDED: vertical
transit, comfort           EXCEEDED: vertical, roll
transit, WFSV practice     OK
light manual work          OK
heavy manual work          OK
intellectual work          OK
```

The example shows the pattern in the text: a sea the vessel can happily transit in can still be too lively for the transfer.

## Caveats

- **RMS is not the peak.** For a typical sea, the largest motions in a record are several times the RMS, so a 3° RMS roll includes occasional rolls well beyond that.
- **Location on the vessel matters.** The transfer limit is quoted in the saloon; accelerations at the bow, where the step-across happens, are higher.
- **Pitch is only given for WFSV transit,** so the code skips it for the other criteria, and my example record has none.

source: Innovative Hull Design for Servicing Offshore Wind TurbinesCORA - page: 67

# Crew Limit Catterman

This is the set of limits the thesis itself applies, so for your notes it supersedes the literature values from the last section: four checks on a stationary service vessel (zero forward speed, pushed up against the turbine).

## The limits (Table 5.4)

| Quantity | Where on vessel | Limit | Basis |
|---|---|---|---|
| MII (motion-induced interruptions) from pitch and heave | Forward perpendicular (bow) | 2–10 events | 0.1–0.5 tips per minute is acceptable [103] |
| RMS vertical acceleration | Forward perpendicular | 0.15g (1.47 m/s²) | Heavy manual work limit [97] |
| RMS vertical acceleration | Centre of gravity (COG) | 0.05g (0.49 m/s²) | Transit passenger limit [97] |
| RMS pitch | COG | 4° | WFSV experience [88] |

## Summary

- **An MII is a loss-of-balance event:** a moment when the motion is enough to make a standing person stumble or grab a handhold and stop what they are doing. It counts discrete events, where RMS measures average intensity.
- **Two locations, two purposes.** The bow is where technicians step across, so it gets the working-on-deck limits. The COG stands in for the passenger saloon.
- **The 0.05g saloon limit is the key predictor.** The thesis notes that this single figure has been found to predict whether a transfer goes ahead [88].
- **Approach is stricter than transit** because people are out on deck. That is why pitch stays at 4° while the vessel-only transit limits from the previous section were looser.
- **Unit mismatch to note:** the text gives MII as a rate (0.1–0.5 per minute) and the table as a count (2–10). They agree if the assessment period is 20 minutes, but that is my inference; check §5.7 for the actual duration.

## Equations

**Vertical acceleration at the bow** from motions at the COG, for a point x_FP metres forward, heave acceleration a_z and pitch angle θ (small angles; the sign depends on the pitch convention):

$$a_{FP}(t) = a_z(t) - x_{FP}\,\ddot{\theta}(t)$$

This is why the bow limit is three times the COG limit: pitch adds a lever-arm term that grows with distance from the centre.

**RMS checks**, as before:

$$\frac{RMS(a_{FP})}{g} \le 0.15 \qquad \frac{RMS(a_z)}{g} \le 0.05 \qquad RMS(\theta) \le 4°$$

**MII tipping condition** (a simplified fore-aft form of the standard criterion, not taken from this thesis). A person with half-stance l and centre-of-gravity height h tips when the horizontal force outweighs what their stance can resist:

$$\left|\,g\sin\theta(t) - \tfrac{l}{h}\,a_{FP}(t)\right| > \tfrac{l}{h}\,g$$

A typical fore-aft value for l/h is about 0.17.

**MII rate from statistics**, if the left-hand side above has RMS σ and mean zero-crossing period T_z:

$$\text{MII per minute} = \frac{60}{T_z}\exp\!\left[-\frac{\left(\tfrac{l}{h}g\right)^2}{2\sigma^2}\right] \text{ per tipping direction}$$

## Code

```python
import numpy as np

G = 9.81
TIP_RATIO = 0.17       # l/h for fore-aft tipping, typical value
LIMITS = {"mii_events": 10, "fp_vertical_g": 0.15, "cog_vertical_g": 0.05, "pitch_deg": 4.0}

def rms(x):
    x = np.asarray(x, dtype=float)
    return np.sqrt(np.mean((x - x.mean()) ** 2))

def bow_acceleration(heave_acc, pitch_rad, dt, x_fp):
    """Vertical acceleration at a point x_fp metres forward of the COG."""
    pitch_acc = np.gradient(np.gradient(pitch_rad, dt), dt)
    return heave_acc - x_fp * pitch_acc

def count_mii(pitch_rad, a_fp):
    """Number of times the tipping threshold is newly exceeded."""
    force = G * np.sin(pitch_rad) - TIP_RATIO * a_fp
    over = np.abs(force) > TIP_RATIO * G
    return int(np.sum(over[1:] & ~over[:-1]))

# Example: 20 minutes at 10 Hz, regular 6 s waves
dt = 0.1
t = np.arange(0, 1200, dt)
heave_acc = 0.8 * np.sin(2 * np.pi * t / 6)                 # m/s^2 at COG
pitch = np.radians(3.0) * np.sin(2 * np.pi * t / 6)         # rad
a_fp = bow_acceleration(heave_acc, pitch, dt, x_fp=10.0)

measured = {
    "mii_events":     count_mii(pitch, a_fp),
    "fp_vertical_g":  rms(a_fp) / G,
    "cog_vertical_g": rms(heave_acc) / G,
    "pitch_deg":      rms(np.degrees(pitch)),
}

for name, value in measured.items():
    status = "OK" if value <= LIMITS[name] else "EXCEEDED"
    print(f"{name:<15} {value:7.3f}  limit {LIMITS[name]:<5}  {status}")
```

Expected output, approximately:

```
mii_events        0.000  limit 10     OK
fp_vertical_g     0.099  limit 0.15   OK
cog_vertical_g    0.058  limit 0.05   EXCEEDED
pitch_deg         2.121  limit 4.0    OK
```

The example is set up to show the thesis's point: the bow and pitch are within limits, but the saloon acceleration is over, and that is the one that stops the transfer.

## Caveats

- **The MII formula is the generic one.** The thesis discusses its own treatment in the sections that follow, so replace mine with theirs when you read §5.7.
- **I've treated 10 events as the ceiling.** The table gives a 2–10 range without saying whether it is an acceptable band or a threshold.
- **Regular waves understate MII.** A sine wave never produces the occasional large motion that causes real stumbles, so the zero count here is an artefact of the toy input; use an irregular sea from the JONSWAP spectrum for a meaningful count.
- **These are limits for one vessel class.** They are applied to catamaran service vessels at zero speed, and other access methods (larger vessels with motion-compensated gangways, helicopters) work to different criteria.

source: Innovative Hull Design for Servicing Offshore Wind TurbinesCORA - page: 179

# Data For Wind & Waves

https://cds.climate.copernicus.eu/datasets/reanalysis-era5-single-levels?tab=download

# Power Curve

I'll look up the paper and the reference turbine data it uses, so the power curves come from the source and not from memory.

The paper (Cheynet et al., 2025, Wind Energy Science) doesn't tabulate the power curves. It plots them in its Figure 5a and cites NREL's public turbine-models repository, so I pulled the three tables from there.

## Power curves

Power in kW at hub-height wind speed, thinned to whole speeds plus the rated point:

| Wind speed (m/s) | NREL 5 MW | IEA 15 MW | NREL 18 MW |
|---|---|---|---|
| 3 | 41 | 70 | 0 |
| 4 | 178 | 595 | 864 |
| 5 | 404 | 1,429 | 1,488 |
| 6 | 738 | 2,656 | 2,728 |
| 7 | 1,187 | 4,339 | 4,584 |
| 8 | 1,771 | 6,481 | 7,057 |
| 9 | 2,519 | 9,229 | 10,146 |
| 10 | 3,448 | 12,661 | 13,852 |
| 11 | 4,563 | 15,000 | 18,000 |
| 12 to 25 | 5,000 | 15,000 | 18,000 |
| above 25 | 0 | 0 | 0 |

| | NREL 5 MW | IEA 15 MW | NREL 18 MW |
|---|---|---|---|
| Hub height | 90 m | 150 m | 156 m |
| Rotor diameter | 126 m | 240 m | 263 m |
| Cut-in speed | 3 m/s | 3 m/s | 4 m/s (first non-zero point) |
| Rated speed | 11.4 m/s | 10.6 m/s | 11 m/s |
| Cut-out speed | 25 m/s | 25 m/s | 25 m/s |

The repository files are finer than this (50 and 60 rows for the first two). The IEA 15 MW file tops out at about 14,995 kW, which I've rounded to rated.

## Why I need it

- **Wind speed is not energy.** A wind dataset gives metres per second; the power curve is the function that converts each speed into kilowatts.
- **The conversion is non-linear, so averages don't work.** Power rises roughly with the cube of speed up to rated, is flat from rated to cut-out, and is zero outside that range. You have to convert every time step and then average. In the example below, plugging the mean wind speed into the curve gives 0.84 where the true capacity factor is 0.61.
- **It completes the O&M chain.** Failure rates and weather windows give downtime in hours; the power curve prices those hours as lost energy, which depends on how windy it was while the turbine was down.

## Equations

**Power curve**, idealised:

$$P(u) = \begin{cases} 0 & u < u_{in} \\ \tfrac{1}{2}\rho A\,C_p(u)\,u^3 & u_{in} \le u < u_{rated} \\ P_{max} & u_{rated} \le u \le u_{out} \\ 0 & u > u_{out} \end{cases}$$

where ρ is air density, A = πD²/4 is the rotor area and C_p is the power coefficient (about 0.48 for these turbines below rated).

**Capacity factor**, the paper's Eq. 2, which it uses because some campaigns were only weeks long:

$$CF = \frac{\overline{P(t)}}{P_{max}} = \frac{1}{N\,P_{max}}\sum_{t=1}^{N} P(u_t)$$

**Annual energy:**

$$E = CF \times P_{max} \times 8760$$

**Lost energy from downtime**, my link to your earlier notes, with D the set of hours the turbine is down:

$$E_{lost} = \sum_{t \in D} P(u_t)$$

## Code

```python
import numpy as np

# Power curves (wind speed m/s, power kW), thinned from NREL's turbine-models repository
CURVES = {
    "NREL 5 MW": dict(rated_kw=5000, hub_m=90,
        u=[3, 4, 5, 6, 7, 8, 9, 10, 11, 11.4, 25],
        p=[40.5, 177.7, 403.9, 737.6, 1187.2, 1771.2, 2518.6, 3448.4, 4562.5, 5000, 5000]),
    "IEA 15 MW": dict(rated_kw=15000, hub_m=150,
        u=[3, 4, 5, 6, 7, 8, 9, 10, 10.6, 25],
        p=[70, 595, 1429, 2656, 4339, 6481, 9229, 12661, 15000, 15000]),
    "NREL 18 MW": dict(rated_kw=18000, hub_m=156,
        u=[3, 4, 5, 6, 7, 8, 9, 10, 11, 25],
        p=[0, 864, 1488, 2728, 4584, 7057, 10146, 13852, 18000, 18000]),
}

def power_kw(u, curve):
    """Power at hub-height wind speed u. Zero below cut-in and above cut-out."""
    u = np.asarray(u, dtype=float)
    return np.interp(u, curve["u"], curve["p"], left=0.0, right=0.0)

def capacity_factor(u, curve):
    """Paper's Eq. 2: mean power over the record divided by rated power."""
    return power_kw(u, curve).mean() / curve["rated_kw"]

# Synthetic hourly hub-height wind speeds for one year (Weibull, mean ~10 m/s)
rng = np.random.default_rng(0)
u = 11.3 * rng.weibull(2.2, size=8760)
print(f"mean wind speed {u.mean():.2f} m/s")

for name, c in CURVES.items():
    cf = capacity_factor(u, c)
    print(f"{name:<11} CF {cf:.2f}   energy {cf * c['rated_kw'] * 8760 / 1e6:6.1f} GWh/yr")

# The mean wind speed alone gives the wrong answer
c = CURVES["IEA 15 MW"]
print(f"power at the mean speed / rated = {power_kw(u.mean(), c) / c['rated_kw']:.2f} "
      f"vs true CF {capacity_factor(u, c):.2f}")
```

Output:

```
mean wind speed 9.97 m/s
NREL 5 MW   CF 0.57   energy   25.0 GWh/yr
IEA 15 MW   CF 0.61   energy   80.5 GWh/yr
NREL 18 MW  CF 0.59   energy   93.3 GWh/yr
power at the mean speed / rated = 0.84 vs true CF 0.61
```

## Caveats

- **The same wind was fed to all three turbines.** In reality the 5 MW hub is at 90 m and sees less wind than the 15 MW at 150 m, which is the main reason their capacity factors differ in Figure 11. Use wind speed at each turbine's own hub height.
- **The wind series is synthetic.** Swap in ERA5 or NORA3 hourly data for a real site.
- **Hub-height speed only.** The paper makes the same simplification and notes that a rotor-averaged speed would be more realistic for rotors this large.
- **Gross output.** No wake, electrical or availability losses are included, so real farm capacity factors are lower.
- **The paper's capacity factors are not site climatology.** Its campaigns were short, and it says the values are only for comparing datasets.

Sources:
- [Cheynet et al. (2025), Tall wind profile validation of ERA5, NORA3, and NEWA datasets using lidar observations, Wind Energy Science 10, 733–754](https://wes.copernicus.org/articles/10/733/2025/)
- [NREL turbine-models repository](https://github.com/NREL/turbine-models)
- [NREL 5 MW reference turbine](https://nrel.github.io/turbine-models/NREL_5MW_126_RWT.html)
- [IEA 15 MW reference turbine](https://nrel.github.io/turbine-models/IEA_15MW_240_RWT.html)
- [NREL 18 MW reference turbine (2020 ATB)](https://nrel.github.io/turbine-models/2020ATB_NREL_Reference_18MW_263.html)

# Vessel speed and capable-vessel wave limit

The speeds are in your screenshot; the wave limits are elsewhere in the paper, so I'll pull those from the article.

The speeds are all in Table 3, but this paper only states a wave height limit for one vessel. Its whole argument is that a fixed wave height limit is too crude, so most vessels are limited by their computed motions instead.

## Vessel speed (Table 3)

| Vessel | Speed (kts) | km/h | Time to cover 100 km |
|---|---|---|---|
| SOV (ROV supported) | 11.2 | 20.7 | 4.8 h |
| AHT, with or without CTV assist | 19.3 | 35.7 | 2.8 h |
| Lead tug, free running | 19.3 | 35.7 | 2.8 h |
| Lead tug, towing a floating turbine | 4.0 | 7.4 | 13.5 h |
| Assist tug / SHC assist tug | 9.8 | 18.1 | 5.5 h |
| SSCV, transit | 8.0 | 14.8 | 6.7 h |
| SSCV, operational | 0.0 | 0 | stationary on site |
| SHC platform, under tow | 6.8 | 12.6 | 7.9 h |

The vessels are:

- **SOV:** service operation vessel.
- **AHT:** anchor handling tug.
- **SSCV:** semi-submersible crane vessel.
- **SHC:** self-hoisting crane, a platform that climbs the turbine tower.

## Wave limits

| Vessel / operation | Limit | Type |
|---|---|---|
| SSCV, transit | Hs up to 4.5 m | Wave height |
| SSCV, component replacement on site | Hs up to 3.5 m | Wave height |
| SHC, hoisting and replacement | Heave ≤ 0.4 m RMS | Motion, not wave height |
| Tugs, AHT and SOV in transit | Accelerations and roll at the vessel's centre of gravity | Motion; values in Tables 4–6 |
| Towed turbine | Accelerations, roll and pitch at the nacelle | Motion; values in Tables 4–6 |

I could not read the bodies of Tables 4–6, which hold the step-by-step limits and durations for each strategy; the page text only gave me their captions. If you send a screenshot of those three tables as you did for Table 3, I'll extract them.

## Equations

**Transit time** in hours, for distance d in km and speed V in knots:

$$t_{transit} = \frac{d}{1.852\,V_s}$$

**Weather window needed** for a tow, which is why the 4.0 kts towing speed matters so much:

$$\tau = t_{transit} + t_{operation}$$

**Fixed-limit check** (the simple approach the paper argues against):

$$\text{OK} = H_s \le H_{lim}$$

**Motion-based check** (the paper's approach), which depends on wave period and heading as well as height, using the response relation from your earlier notes:

$$x_{RMS}(H_s, T_p, \text{heading}) = \sqrt{\int_0^\infty |RAO(f)|^2\,S(f)\,df} \;\le\; L$$

The same 3 m sea can pass or fail depending on its period and direction.

## Code

```python
KTS_TO_KMH = 1.852

SPEED_KTS = {
    "SOV": 11.2,
    "AHT": 19.3,
    "Lead tug (free)": 19.3,
    "Lead tug (towing FOWT)": 4.0,
    "Assist tug": 9.8,
    "SSCV (transit)": 8.0,
    "SHC platform (towed)": 6.8,
}

# Only the limits the paper states as wave heights
HS_LIMIT_M = {
    "SSCV (transit)": 4.5,
    "SSCV (on-site replacement)": 3.5,
}

def transit_hours(distance_km, vessel):
    return distance_km / (SPEED_KTS[vessel] * KTS_TO_KMH)

def can_operate(hs, operation):
    return hs <= HS_LIMIT_M[operation]

for vessel in SPEED_KTS:
    print(f"{vessel:<24} {transit_hours(100, vessel):5.1f} h per 100 km")

print(can_operate(4.0, "SSCV (transit)"))              # True
print(can_operate(4.0, "SSCV (on-site replacement)"))  # False
```

## Caveats

- **Towing is the bottleneck.** At 4.0 kts a 100 km tow takes over half a day each way, so tow-to-port needs a long calm window that crane-on-site strategies don't.
- **These are heavy-maintenance vessels.** The limits are for major component replacement on floating turbines, not routine technician transfer, so they don't replace the 1.5 m crew transfer figure in your earlier notes.
- **The paper's site distances** are not in the table, so I used a round 100 km; substitute the real port distance.

Sources:
- [Dighe et al. (2024), Improving O&M Simulations by Integrating Vessel Motions for Floating Wind Farms, J. Mar. Sci. Eng. 12(11), 1948](https://www.mdpi.com/2077-1312/12/11/1948)