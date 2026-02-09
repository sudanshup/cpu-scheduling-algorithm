# Building a Thermal-Aware CPU Scheduler: A Machine Learning Implementation Guide

**Version:** 1.0  
**Based on:** Analysis of Dataset (50,863 samples, "Witcher 3" Workload)  
**Target System:** Multi-Core Processor (4 Cores detected)  

---

## 1. Executive Summary & Problem Definition

Traditional CPU schedulers (like CFS in Linux or Windows Scheduler) prioritize **fairness** and **throughput**. However, your dataset analysis reveals a critical hardware asymmetry: **Core 0 runs 1.5°C hotter and consumes 2.2x more power** than Core 3 for the same workload.

This guide details how to build a Machine Learning model that transitions from *Load Balancing* to *Thermal Balancing*.

**The Objective:**
Build a scheduling agent that assigns high-intensity processes (like `witcher3`) to the most thermally efficient cores (Core 3) to minimize aggregate system temperature without throttling performance.

---

## 2. Feature Engineering (From Your Data)

Before selecting an algorithm, we must transform your raw telemetry into training features. The dataset analysis highlights specific correlations that dictate which features matter.

### 2.1 Input Features ($X$)
These features feed into the model to describe the current "State."

| Feature Name | Data Source | Rationale from Analysis |
| :--- | :--- | :--- |
| **`Core_ID`** | System | **Critical.** Core 0 and Core 3 behave differently. The model must learn this hardware bias. |
| **`Current_Core_Temp`** | Sensors | Base state. Your data ranges 52°C - 88°C. |
| **`Power_Draw_W`** | Sensors | **Strong Predictor.** Core 0 draws ~4.1W vs Core 3 ~1.9W. |
| **`Process_CPU_Load`** | OS API | `witcher3` (382%) vs `brave` (6%). Measures "Heat Potential." |
| **`Thermal_Velocity`** | Derived | Rate of change ($T_{now} - T_{t-5s}$). Identifies rapid spikes. |
| **`Daytime_Running`** | Derived | Time since boot. Captures heat soak/saturation over time. |

### 2.2 Target Variables ($Y$)
Depending on the algorithm chosen, we are trying to predict different things.

* **Regression Target:** `Next_Temperature_5s` (Predicting the future temp).
* **Classification Target:** `Optimal_Core_Index` (0, 1, 2, or 3).



---

## 3. Algorithm Selection Strategy

Based on the dataset's characteristics (time-series data, thermal inertia, non-linear power scaling), here are the three recommended approaches tailored to your complexity requirements.

### Approach A: The "Predictive" Model (Time-Series Regression)
**Algorithm:** **LSTM (Long Short-Term Memory)** or **GRU (Gated Recurrent Unit)**

* **Why this fits your data:** Your analysis showed a "Feedback Loop" (Heat → Throttle → Load shift). Standard algorithms treat data points as independent, but thermal data is *sequential*. A high temp at $t=0$ strongly influences $t+1$.
* **Mechanism:** The model looks at the last 10 snapshots (50 seconds) and predicts: *"If I place `witcher3` on Core 0, the temp will hit 89°C in 20 seconds."*
* **Pros:** Highly accurate for thermal inertia.
* **Cons:** Computationally expensive to run every 5 seconds.

### Approach B: The "Reactive" Model (Decision Trees)
**Algorithm:** **XGBoost (Gradient Boosted Regressor)** or **Random Forest**

* **Why this fits your data:** You have distinct categories of processes. `witcher3` is a distinct outlier compared to `System` processes. Tree-based models are excellent at isolating these outliers and creating split-points (e.g., *IF CPU > 100% AND Core_Temp > 80°C THEN...*).
* **Mechanism:** Predicts a "Thermal Cost" score. The scheduler places tasks on the core with the lowest predicted cost.
* **Pros:** Fast inference (<1ms), interpretable, handles non-linear power curves well.
* **Cons:** Doesn't "remember" history (no temporal awareness).

### Approach C: The "Adaptive" Model (Reinforcement Learning)
**Algorithm:** **Deep Q-Network (DQN)** or **PPO (Proximal Policy Optimization)**

* **Why this fits your data:** The goal is optimization over time. RL agents learn by *doing*. The agent will punish itself for placing tasks on Core 0 (receiving a negative reward due to high power/heat) and learn to favor Core 3 naturally.
* **Mechanism:**
    * **State:** Current Temps + Queue.
    * **Action:** Assign Process $P$ to Core $C$.
    * **Reward:** $+1$ for low temp, $-10$ for throttling, $-5$ for high power.
* **Pros:** Self-optimizing. It will discover that Core 0 is inefficient without you hard-coding it.
* **Cons:** Complex to train; requires a simulated environment first.



[Image of reinforcement learning cycle]


---

## 4. Recommended Implementation: The "Hybrid" Approach

For a realistic CPU scheduler, a pure Deep Learning model is often too slow. We recommend a **Hybrid Random Forest + PID Controller** approach.

### Step 1: Pre-Classification (The Classifier)
Use a **Random Forest Classifier** to bucket processes based on their thermal signature (derived from your `ProcessType` analysis).

* **Class A (Inferno):** `witcher3`, Video Encoders (>100% CPU).
* **Class B (Warm):** `brave`, `electron` apps (10-50% CPU).
* **Class C (Cool):** `System`, `notepad` (<10% CPU).

### Step 2: The Core Scorer (The Regressor)
Use a lightweight **Linear Regressor** updated in real-time to calculate a "Suitability Score" for each core.

$$Score_{core} = w_1(T_{max} - T_{current}) + w_2(\text{InversePower})$$

* *Based on your data, Core 3 will naturally have a higher score because $T_{current}$ is lower and Power is lower.*

### Step 3: The Scheduler Logic
```python
def schedule_task(process):
    # 1. Classify Process
    p_type = rf_model.predict(process.features)
    
    # 2. Get Core Scores
    scores = calculate_core_scores() 
    # (Core0: 20, Core1: 50, Core2: 55, Core3: 80)
    
    if p_type == "Inferno":
        # Critical Logic: "Inferno" tasks ONLY go to the best cores
        assign_to(process, best_core=Core3)
    elif p_type == "Cool":
        # Packing Logic: Put cool tasks on hot cores to save cool cores for gaming
        assign_to(process, worst_acceptable_core=Core0)
```

## 5. Training The Model (Python Workflow)
```python
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split

# 1. Load Data
df = pd.read_csv('cpu_thermal_data.csv')

# 2. Label Engineering
# We define "High Thermal Impact" as any process that pushed Temp > 80°C
# This allows the model to learn WHICH processes are dangerous.
df['Is_Thermal_Risk'] = (df['CPU_Percent'] > 50) | (df['WorkingSet_MB'] > 1000)

# 3. Feature Selection
features = ['CPU_Percent', 'WorkingSet_MB', 'ThreadCount', 'BasePriority']
X = df[features]
y = df['Is_Thermal_Risk']

# 4. Train Model
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
clf = RandomForestClassifier(n_estimators=100)
clf.fit(X_train, y_train)

# 5. Extract Insights
print("Feature Importance:")
for name, importance in zip(features, clf.feature_importances_):
    print(f"{name}: {importance:.4f}")
```
