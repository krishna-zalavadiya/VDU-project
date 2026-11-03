# Data Analysis for Network Traffic (CICIDS-2017)

**Purpose**  
Build an end-to-end **intrusion detection analytics pipeline** that transforms raw network traffic into **clean features**, evaluates multiple **ML classifiers** (LogReg, DT, RF, XGBoost, GNB), performs **dimensionality reduction** via PCA (accuracy vs. speed trade-offs), and **simulates real-time** monitoring by streaming predictions to **Power BI** dashboards.

**What this showcases**  
- **Data Engineering**: robust preprocessing (null/∞ handling, label encoding, leakage-safe scaling)  
- **Data Analysis**: EDA, correlation analysis, attack distributions, protocol analysis  
- **Machine Learning**: multi-model training & evaluation; confusion matrices; feature importance; metrics comparison  
- **Optimization**: **PCA** variance thresholds → **speed vs. accuracy** decisions for real-time scenarios  
- **Visualization**: Power BI **live dashboards** from a Python streaming pipeline  
- **Reproducibility**: notebook-driven workflow with saved artifacts (scalers/models)

**Key Outcomes**  
- **Rigorous model comparison**: Random Forest achieved top raw accuracy (**99.94%**), while XGBoost was selected for production (**99.93%**) due to **100× lower memory consumption** (24 MB vs. 2.5 GB) and low-latency inference.  
- **PCA @ 0.99 variance**: Compresses 78 features to 33 components, yielding a **71% faster training/inference speedup** with only a **0.016%** accuracy delta for real-time simulation.  
- **Operational flow**: Raw CSVs → leakage-free engineering → trained models → live Python streaming → Power BI dashboards.

----
</br>

<!-- Hero visuals (from your repo's /images folder) -->
<p align="center">
  <img src="images/Distribution of Attack Type in CICIDS 2017 Dataset.png" alt="Attack distribution" width="48%" />
  <img src="images/Correlation Heatmap between attributes.png" alt="Correlation heatmap" width="48%" />
</p>

----

## System Overview

**Flow**
1) **Ingest** CICIDS-2017 daily CSVs  
2) **Preprocess**: drop IDs/IPs/timestamps → encode label → handle null/∞ → train/test split → fit **StandardScaler on train only**  
3) **Model**: train LR, DT, RF, XGB, GNB → evaluate (Accuracy, Precision, Recall, F1, MCC) → confusion matrices  
4) **Optimize**: evaluate **PCA** at variance thresholds (0.90/0.95/0.99/0.999) → select trade-off for real-time  
5) **Simulate real-time**: Python loop → preprocess + predict → POST to **Power BI** Streaming Dataset  
6) **Visualize**: live tiles & reports (daily trends, flagged events, heatmaps, KPIs)

### onceptual Diagram

```mermaid
flowchart LR
  A[CICIDS-2017 CSVs] --> B[Preprocessing: drop IDs and IPs, encode label, handle null and infinite, split, scale]
  B --> C[Model Training: LR, DT, RF, XGB, GNB]
  C --> D[Evaluation: metrics, confusion matrices, feature importance]
  D --> E[PCA Trade-offs: 0.90, 0.95, 0.99, 0.999]
  E --> F[Real-Time Simulation: Python streaming predictions]
  F --> G[Power BI Streaming Dataset]
  G --> H[Live Dashboards: KPIs, trends, heatmaps]
```

### Why This Design?

- Keeps **data leakage** out by scaling *after* the split  
- Compares **classical ML models** for interpretable baselines and production-friendly performance  
- Uses **PCA** only where it helps latency, not blindly everywhere  
- Separates **batch evaluation** from **streaming simulation** to mirror real ops  

---

<!-- Optional second visual for architecture section -->
<p align="center">
  <img src="images/Accuracy of Different Model.png" alt="Accuracy across models" width="60%" />
</p>

---

## 2) Dataset & Challenges

**Dataset: [CICIDS-2017](https://www.unb.ca/cic/datasets/ids-2017.html)**  
- One of the most widely used **Intrusion Detection System (IDS)** benchmarks.  
- Traffic captured over 5 days in 2017 at the University of New Brunswick.  
- **Mix of benign and malicious flows**: DDoS, PortScan, Botnet, Infiltration, Heartbleed, etc.  
- **80+ features**: packet counts, byte rates, flags, durations, header stats.  

### Key Properties
- **Multi-class labels** (`BENIGN` + multiple attack types)  
- **Highly imbalanced**: e.g., some attack types dominate, others are rare  
- **Mixed data types**: integers, floats, categorical (protocols)  
- **Dirty fields**: nulls, “Infinity” in rate features, non-informative IDs (Flow ID, IP addresses)  

---

### Distribution Examples

**Attack distribution**
<p align="center">
  <img src="images/Distribution of Attack Type in CICIDS 2017 Dataset.png" alt="Distribution of attack types" width="70%" />
</p>

**Protocol counts**
<p align="center">
  <img src="images/Count of Protocol.png" alt="Protocol counts" width="45%" />
  <img src="images/Count of Label across various Protocol.png" alt="Label counts across protocols" width="45%" />
</p>

---

### Data Cleaning Challenges

1. **Irrelevant IDs**  
   - `Flow ID`, `Source IP`, `Destination IP`, `Timestamp` → removed  
   - Why? They don’t generalize for detection, and may cause leakage.  

2. **Missing Values**  
   - Features like `Flow Bytes/s` had **nulls** → imputed (safe defaults / median).  

3. **Infinite Values**  
   - Rate-based features sometimes yielded **∞** (e.g., division by zero).  
   - Replaced with capped values / imputed safely.  

4. **Skewed Classes**  
   - Some attacks extremely under-represented.  
   - Approached as a **binary detection problem (Benign vs Malicious)** first.  

5. **Mixed Data Types**  
   - Encoded labels (`BENIGN=0`, `ATTACK=1`).  
   - Continuous features scaled using **StandardScaler** (fit only on train split).  

---

### Column Type Mix

<p align="center">
  <img src="images/Distribution of Column Types.png" alt="Distribution of column types" width="60%" />
</p>


---

## 3) Data Engineering Pipeline

Turning CICIDS-2017 raw CSVs into **ML-ready features** required a robust preprocessing pipeline.  
Every transformation was chosen to **improve generalization, avoid leakage, and stabilize training**.

---

### Step-by-Step Transformations

1. **Concatenate Daily Files**  
   - CICIDS-2017 is split across multiple CSVs (per day).  
   - Merged into a **single unified dataset** for consistency.  

2. **Drop Identifiers & Non-Informative Fields**  
   - Removed: `Flow ID`, `Source IP`, `Destination IP`, `Timestamp`.  
   - *Reason*: These carry unique/session-specific values, not predictive patterns. Keeping them risks **data leakage**.

3. **Label Encoding**  
   - Converted `Label` → **binary classification** (`0 = Benign`, `1 = Attack`).  
   - *Reason*: Models perform better with numeric encoding, and binary framing avoids extreme class imbalance.

4. **Missing Value Handling**  
   - Columns like `Flow Bytes/s`, `Flow Packets/s` contained **NaNs**.  
   - Imputed using **median values** (robust against skew).  

5. **Infinity Replacement**  
   - Some flow features (rates, divisions) produced **∞** values.  
   - Strategy: Replace with column maximum or safe constant.  
   - *Reason*: Avoids math errors and ensures consistency.

6. **Train-Test Split (80/20)**  
   - Ensured **random shuffling** while preserving label balance.  
   - *Reason*: Avoid contamination of train/test.

7. **Scaling with StandardScaler**  
   - Fit **only on training data**, then applied to test.  
   - *Reason*: Prevents data leakage (test info influencing scaling).  
   - Scaling critical since models like Logistic Regression and XGBoost are sensitive to feature magnitudes.

---

### Preprocessing Pipeline (Conceptual)

```mermaid
flowchart TD
  A[Raw CICIDS-2017 CSVs] --> B[Concatenate Files]
  B --> C[Drop IDs and IPs]
  C --> D[Encode Labels]
  D --> E[Handle NaN and Infinite Values]
  E --> F[Train Test Split]
  F --> G[Fit StandardScaler on Train]
  G --> H[Transform Train and Test Features]
```

### Why This Matters

- **Leakage-safe scaling** → ensures real-world robustness  
- **Consistent handling of ∞ & NaN** → prevents crashes during streaming simulation  
- **Binary framing** → provides a strong first baseline; later extensible to multi-class attacks  
- **Reproducibility** → scaler & models saved under `Packages/` and `Models/` for reuse  

---

## 4) Exploratory Data Analysis (EDA)

Before modeling, we **interrogated the dataset** to uncover signal, bias, and anomalies.  
This ensured that models were trained on **meaningful, stable features** — not noise.

---

### 4.1 Distribution of Attack Types
- Dataset is **heavily imbalanced** — DDoS and PortScan dominate.  
- Rare classes (e.g., Heartbleed, Infiltration) appear only a handful of times.  
- Insight: Without rebalancing, models risk **bias towards majority classes**.  
- Solution: Start with **binary detection** (Benign vs. Attack) → reduces skew.

<p align="center">
  <img src="images/Distribution of Attack Type in CICIDS 2017 Dataset.png" alt="Distribution of Attack Types" width="70%" />
</p>

---

### 4.2 Flow Duration Patterns
- **Box plots** show how attack types differ in session length.  
- Example: **DDoS flows** are very short; **Infiltration flows** are long-lived.  
- *Interpretation*: Flow duration can be a **strong feature** for discriminating attacks.

<p align="center">
  <img src="images/Box Plot of Flow Duration for Various Attack Types.png" alt="Flow Duration Box Plot" width="70%" />
</p>

---

### 4.3 Daily Trends
- Averaged flow duration per day shows **temporal variability**.  
- Malicious traffic bursts occur on specific days → simulates **real-world attack windows**.

<p align="center">
  <img src="images/Average Flow Duration for Each Day.png" alt="Average Flow Duration Per Day" width="70%" />
</p>

---

### 4.4 Protocol & Flag Analysis
- Protocol usage skews: TCP dominates, UDP less frequent.  
- Attack traffic often manipulates flag distributions (SYN floods, abnormal flag counts).  
- **Flag counts by label** → reveals systematic **signature differences**.

<p align="center">
  <img src="images/Average of Various Flag Count for each Label.png" alt="Flag Count vs Label" width="70%" />
</p>

---

### 4.5 Feature Correlations & Density Distributions
- High correlation among rate-based features → risk of **multicollinearity**.  
- Correlation heatmap confirms redundancies that PCA later exploits.  
- Kernel Density Estimates (KDE) and Violin plots highlight distinct distribution profiles between Benign traffic and malicious spikes in byte rates.

<p align="center">
  <img src="images/Correlation Matrix between Attributes.png" alt="Correlation Matrix" width="48%" />
  <img src="images/Correlation Heatmap between attributes.png" alt="Correlation Heatmap" width="48%" />
</p>
<p align="center">
  <img src="images/Byte Rate KDE Distributions.png" alt="Byte Rate KDE Distributions" width="48%" />
  <img src="images/Feature Violin Plots.png" alt="Feature Violin Plots" width="48%" />
</p>

---

### EDA Takeaways
- **Severe imbalance** → justify binary framing & careful evaluation metrics  
- **Certain features (duration, flags, byte rates)** carry strong predictive signal  
- **Redundancies in features** → motivates PCA for dimensionality reduction  
- **Temporal attack bursts** → simulation of “real-world-like” traffic feasible

---

## 5) Modeling Methodology

We designed a **systematic ML benchmarking study** on CICIDS-2017.  
The goal: compare **classical ML algorithms** with different **bias-variance trade-offs, interpretability, and computational efficiency**.

---

### 5.1 Models Selected & Rationale

| Model                   | Why It Was Chosen                                                   | Strengths                         | Weaknesses                           |
|--------------------------|---------------------------------------------------------------------|-----------------------------------|---------------------------------------|
| **Logistic Regression** | Simple linear baseline, interpretable coefficients                 | Transparent, fast, probabilistic  | Limited to linear decision boundaries |
| **Decision Tree**       | Nonlinear splits, interpretable rules                               | Easy to visualize, no scaling req | High variance (overfitting risk)      |
| **Random Forest**       | Ensemble of trees → reduces variance                                | Strong generalization, robust     | Slower training, less interpretable   |
| **XGBoost**             | Gradient boosting, regularized, highly competitive for tabular data | Excellent accuracy, handles skew  | More tuning required, less transparent|
| **Gaussian Naive Bayes**| Probabilistic baseline, fast                                        | Handles categorical-like features | Assumes feature independence          |

📌 **Key Idea:** Cover the spectrum — from interpretable (LogReg, DT) to ensemble (RF, XGB) to lightweight baseline (GNB).

---

### 5.2 Experimental Protocol

1. **Split:** 80/20 train-test  
2. **Scaling:** StandardScaler applied where needed (LR, GNB, XGB)  
3. **Metrics:** Accuracy, Precision, Recall, F1-score, Matthews Correlation Coefficient (MCC)  
   - *Reason for MCC*: handles class imbalance better than accuracy alone  
4. **Confusion Matrices:** to visualize **false positives (FP)** vs **false negatives (FN)** trade-offs  
5. **Repeatability:** Each model + scaler object saved under `Packages and Models/`

---

### 5.3 Results: Comprehensive Benchmark Across Models

Below is the verified performance benchmark on the CICIDS-2017 test set (639,695 flows, 20% holdout):

| Model | Accuracy | Precision (Weighted) | Recall (Weighted) | F1-Score | MCC | ROC-AUC | Memory (MB) | Training Time (s) |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Random Forest** | **99.94%** | **0.9994** | **0.9994** | **0.9994** | **0.9984** | **0.9999** | 2,484.5 MB | 577.3 s |
| **XGBoost** | **99.93%** | **0.9993** | **0.9993** | **0.9993** | **0.9981** | **0.9998** | **24.4 MB** | **520.8 s** |
| **Decision Tree** | 99.90% | 0.9990 | 0.9990 | 0.9990 | 0.9975 | 0.9989 | 900.5 MB | 287.9 s |
| **Logistic Regression** | 97.71% | 0.9784 | 0.9771 | 0.9769 | 0.9424 | 0.9979 | 1,991.5 MB | 1,331.1 s |
| **Gaussian Naive Bayes**| 86.98% | 0.9474 | 0.8698 | 0.8976 | 0.7513 | 0.9260 | 2,357.6 MB | 7.1 s |

<p align="center">
  <img src="images/Accuracy of Different Model.png" alt="Accuracy comparison across models" width="55%" />
  <img src="images/Model Performance Radar Chart.png" alt="Radar chart comparison" width="42%" />
</p>

<p align="center">
  <img src="images/ROC Curves All Models.png" alt="ROC Curves across all models" width="48%" />
  <img src="images/PR Curves All Models.png" alt="Precision-Recall curves" width="48%" />
</p>

---

### 5.4 Confusion Matrices (Error Profiles)

Confusion matrices reveal *how* errors are distributed — critical in IDS, where **false negatives (missed attacks)** are far more dangerous than false positives.

<p align="center">
  <img src="images/Confusion Matrix of Logistic Regression.png" alt="LogReg Confusion" width="45%" />
  <img src="images/Confusion Matrix of Decision Tree Classifier.png" alt="DT Confusion" width="45%" />
</p>
<p align="center">
  <img src="images/Confusion Matrix of Random Forest Classifier.png" alt="RF Confusion" width="45%" />
  <img src="images/Confusion Matrix of XGBoost Classifier.png" alt="XGB Confusion" width="45%" />
</p>
<p align="center">
  <img src="images/Confusion Matrix of Gaussian Naive Bayes.png" alt="GNB Confusion" width="45%" />
</p>

**Observations**
- **Random Forest**: Achieved the highest raw accuracy (99.94%), with near-zero false negatives across all classes, including rare intrusions.
- **XGBoost**: Virtually identical detection capability (99.93%) with lowest latency and minimal false alarms.
- **Decision Tree**: Strong baseline (99.90%) with depth regularization (`max_depth=20`) to prevent over-memorization.
- **Logistic Regression**: Linear boundary limitations yield lower recall on complex attacks and higher training time (1,331s) due to convergence over high-dimensional space.
- **Naive Bayes**: Weaker F1 (89.76%) due to violated feature independence assumptions on correlated network packet rates.

---

### 5.5 Production Selection: Why XGBoost Over Random Forest?

While **Random Forest achieved the highest raw accuracy (99.94% vs. 99.93%)**, **XGBoost was chosen as the operational deployment model** for critical engineering reasons:
1. **100× Lower Memory Footprint:** Random Forest required **2,484.5 MB (~2.5 GB)** of active RAM to hold unpruned tree ensembles across 3.2M records, compared to just **24.4 MB for XGBoost**.
2. **Compact Serialized Model:** XGBoost compiles into a tight **1.6 MB** binary (`xgb_model.model`), allowing instant loading in resource-constrained IDS sensors and edge gateways.
3. **PCA Synergy & Low Latency:** When compressed via PCA (0.99 variance), XGBoost yields a **71% training and inference speed-up** with only a 0.016% accuracy drop, making it the ideal engine for real-time streaming to Power BI.

---

## 6) Feature Importance & Interpretability

High predictive accuracy alone isn’t enough in **network intrusion detection**.  
We need to know **which features drive predictions** — for transparency, debugging, and trust in deployment.

---

### 6.1 Logistic Regression Coefficients
- Linear model → coefficients show **direction & magnitude** of feature impact.  
- Example: Longer **Flow Duration** increases probability of malicious label.  
- Useful for **interpretable baselines** and feature sanity-checks.  

<p align="center">
  <img src="images/Feature Importance from Logistic Regression.png" alt="LogReg Coefficients" width="60%" />
</p>

---

### 6.2 Decision Tree Importance
- Splits highlight **which features best separate benign vs attack traffic**.  
- Provides **rule-based intuition** (e.g., “if flow duration < threshold → benign”).  

<p align="center">
  <img src="images/Feature Importance from Decision Tree.png" alt="DT Importance" width="60%" />
</p>

---

### 6.3 Random Forest Importance
- Aggregates many trees → robust measure of feature relevance.  
- Helps reduce overfitting bias from single trees.  
- Confirms consistency: flow duration, byte rates, flag counts rank highly.  

<p align="center">
  <img src="images/Feature Importance from Random Forest.png" alt="RF Importance" width="60%" />
</p>

---

### 6.4 XGBoost Importance (Winner Model)
- Provides **multiple importance metrics** (Gain, Cover, Frequency).  
- **Gain** shows which features most improve split quality.  
- Here, **Flow Duration, Destination Port, Packet Length Mean** dominate.  
- Security Insight: Attack flows often exploit **specific ports** and **abnormal packet lengths**.

<p align="center">
  <img src="images/Feature Importance from XGBoost.png" alt="XGB Importance" width="60%" />
</p>

---

### Takeaways
- **Flow Duration** is consistently critical → many attacks differ in session length.  
- **Destination Port** patterns emerge in XGBoost → port-based exploits common.  
- **Packet Length Mean / Byte Rates** strongly influence detection → consistent with flooding attacks.  
- Feature importance analysis validates that models are not just memorizing noise — they’re leveraging meaningful network properties.
---

## 7) PCA & Performance Optimization

With >80 features, many of them highly correlated (see heatmaps), we explored **Principal Component Analysis (PCA)** to reduce dimensionality.

---

### 7.1 Why PCA?
- Many CICIDS-2017 features are **redundant** (correlated rates, byte counts).  
- High dimensionality increases:
  - **Training time** (slower iteration cycles)  
  - **Risk of overfitting**  
  - **Deployment latency** (bad for real-time IDS)  
- PCA can compress features into a smaller set of **orthogonal components** that capture most variance.

---

### 7.2 Variance Threshold Experiments

To identify the optimal trade-off between dimensionality reduction and classification accuracy, we evaluated PCA across four variance retention thresholds:

| Variance Threshold | Number of Components | Retained Features | Accuracy (%) | Training Time (s) | Memory (MB) |
| :---: | :---: | :---: | :---: | :---: | :---: |
| **0.90 (90%)** | 21 | 21 / 78 (73% compression) | 99.29% | 123.3 s | 24.4 MB |
| **0.95 (95%)** | 26 | 26 / 78 (67% compression) | 99.89% | 124.2 s | 24.4 MB |
| **0.99 (99%)** | **33** | **33 / 78 (58% compression)** | **99.91%** | **154.2 s** | **24.4 MB** |
| **0.999 (99.9%)** | 42 | 42 / 78 (46% compression) | 99.89% | 169.3 s | 24.4 MB |

<p align="center">
  <img src="images/PCA Scree Plot.png" alt="PCA Scree Plot" width="48%" />
  <img src="images/PCA 2D Class Separability.png" alt="PCA 2D Class Separability" width="48%" />
</p>

📊 Impact on metrics across thresholds:

<p align="center">
  <img src="images/Impact of Variance Threshold on Metrics.png" alt="PCA Variance Threshold Impact" width="70%" />
</p>

---

### 7.3 Performance Metrics Across Thresholds

<p align="center">
  <img src="images/PERFORMANCE METRICS FOR VARIANCE THRESHOLD.png" alt="PCA Performance Metrics" width="70%" />
</p>

---

### 7.4 With vs Without PCA: Direct Comparison

| Metric | Without PCA (Full Features) | With PCA (0.99 Variance) | Delta / Benefit |
| :--- | :---: | :---: | :---: |
| **Input Features** | 78 features | **33 principal components** | **58% feature reduction** |
| **Accuracy** | 99.93% (`0.99927`) | 99.91% (`0.99911`) | **Only 0.016% drop** |
| **Training Time** | 520.8 s | **151.0 s** | **71.0% faster training!** |
| **Memory Footprint** | 24.4 MB | 24.4 MB | Lightweight & stable |

<p align="center">
  <img src="images/Peformance Metrics Comparison with and without PCA.png" alt="PCA vs No PCA Comparison" width="70%" />
</p>

---

### Takeaway
- **PCA @ 0.99 variance is the definitive sweet spot**: compresses 78 features down to **33 components**, delivers a **71% reduction in training latency**, with virtually zero performance penalty (**<0.02% accuracy difference**).
- **Production Architecture**:
  - **Batch Analytics & Auditing**: XGBoost without PCA for exhaustive feature interpretation.
  - **Live IDS Streaming Simulation**: XGBoost with PCA @ 0.99 for real-time low-latency packet flow classification into Power BI.

---

## 8) Real-Time Simulation Design

After training and validating models, we built a **simulation pipeline** to mimic how an Intrusion Detection System (IDS) would run in production.  
Instead of just reporting static metrics, the system generates **real-time predictions** and pushes them into a **Power BI Streaming Dataset** for live dashboards.

---

### 8.1 Operational Flow

1. **Sample Traffic Flow**
   - Randomly selects records from the CICIDS-2017 dataset
   - Mimics live arrival of new traffic events

2. **Preprocessing**
   - Apply the same pipeline: drop IDs → impute → scale → (PCA at 0.99 variance)

3. **Prediction**
   - Run through **XGBoost model** (winner from benchmarking)
   - Classify as `Benign (0)` or `Attack (1)`

4. **Stream to Power BI**
   - Use Python script to **POST results to Power BI REST API**
   - Power BI stores records in a **Streaming Dataset**

5. **Live Dashboards**
   - Visual tiles auto-refresh with near real-time latency
   - KPIs, daily trends, attack heatmaps, and alert counts visible instantly

---

### 8.2 Flow Diagram

```mermaid
flowchart TD
    A[Sampled Flow from CICIDS Dataset] --> B[Preprocessing Pipeline]
    B --> C[XGBoost Prediction]
    C --> D[POST JSON Payload to Power BI API]
    D --> E[Power BI Streaming Dataset]
    E --> F[Real-Time Dashboard: KPIs, Trends, Alerts]
```

### 8.3 Example JSON Payload (to Power BI API)
```json
{
  "flow_id": "123456",
  "duration": 2.13,
  "protocol": "TCP",
  "prediction": "Attack",
  "timestamp": "2025-09-29T12:34:56Z"
}
```
---

### 8.4 Dashboard Screens

**Realtime Streaming Flow**

---
<p align="center"> <img src="images/Average of Various Flag Count for each Label.png" alt="Realtime Flow Simulation" width="65%" /> </p>

---

**Daily Trends of Attack Traffic**

---

<p align="center"> <img src="images/Average Flow Duration for Each Day.png" alt="Daily Trends Dashboard" width="70%" /> </p>

---

**Power BI Live Dashboard**

---

<p align="center"> <img src="images/Count of Protocol.png" alt="Power BI Dashboard Example" width="45%" /> <img src="images/Count of Label across various Protocol.png" alt="Power BI Dashboard Example 2" width="45%" /> </p>

---

### Takeaway

- Built a true end-to-end pipeline: from offline ML → real-time prediction → streaming visualization.

- Demonstrates skills in data engineering + ML + visualization integration.

- Mirrors production-grade IDS workflows: detection → alerting → monitoring.

---

## 9) Visualization Layer & Insights

The final layer of the pipeline is **business-facing visualization**.  
While preprocessing + ML happen in the backend, decision-makers interact with **Power BI dashboards** that continuously update from the **streaming dataset**.

---

### 9.1 Dashboard Goals
- Provide **network operators** with **live insights** into:
  - Volume of benign vs malicious flows
  - Daily / hourly traffic trends
  - Protocol distribution (TCP, UDP, ICMP)
  - Attack-type breakdowns
  - Alerts for flagged anomalies

---

### 9.2 Example Dashboards

**1. Daily Attack Trends**
- Highlights bursts of malicious activity
- Helps identify **attack windows** (e.g., DDoS peaks)

<p align="center">
  <img src="images/Average Flow Duration for Each Day.png" alt="Daily Attack Trends" width="70%" />
</p>

---

**2. Flagged Records & Alerts**
- Displays live count of **benign vs attack flows**
- Critical for **real-time alerting**

<p align="center">
  <img src="images/Average of Various Flag Count for each Label.png" alt="Flag Counts by Label" width="70%" />
</p>

---

**3. Protocol & Label Distributions**
- Breakdown of flows across **TCP, UDP, ICMP**
- Shows how attacks target different protocols

<p align="center">
  <img src="images/Count of Protocol.png" alt="Protocol Distribution" width="45%" />
  <img src="images/Count of Label across various Protocol.png" alt="Label Distribution by Protocol" width="45%" />
</p>

---

**4. Correlation & Heatmaps**
- **Visual EDA** included in dashboard
- Helps analysts spot redundant features and relationships

<p align="center">
  <img src="images/Correlation Heatmap between attributes.png" alt="Correlation Heatmap" width="65%" />
</p>

---

### 9.3 Live Demo (Power BI in Action)

**Power BI Realtime Demo — Without PCA**
<p align="center">
  <img src="images/powerbi_demo_without_pca.gif" alt="Power BI Realtime Demo — without PCA" width="80%" />
</p>

---

**Power BI Realtime Demo — With PCA**
<p align="center">
  <img src="images/powerbi_demo_with_pca.gif" alt="Power BI Realtime Demo — with PCA" width="80%" />
</p>


---

### 9.4 Operator Insights
- **Real-time situational awareness**: dashboards update instantly from the stream  
- **Explainable ML integration**: confusion matrices + feature importance plots included in reports  
- **Business alignment**: KPIs and alerts framed for SOC (Security Operations Center) workflows  

---

### Takeaway
The visualization layer transforms **raw predictions** into **actionable insights**:  
- Analysts can **spot anomalies quickly**  
- Security teams gain **trust** through transparent metrics (false positives vs false negatives)  
- Decision-makers see a **real-time, live operational view**

---

## 10) Reproducibility & Environment Setup

This repo is structured for **reproducible experiments** and **easy simulation**.  
Every folder has a clear role, and environment dependencies are pinned.

---

### 10.1 Repository Structure

```text
Data-Analysis-for-Network-Traffic-Analysis/
│
├── Data/                          # CICIDS-2017 CSVs (external, download required)
├── Packages and Models/            # Saved scalers, trained models (pickle files)
├── Python File/                    # Jupyter notebooks (preprocessing, training, PCA, streaming)
├── images/                         # Figures (EDA plots, confusion matrices, feature importance, PCA charts)
├── power_bi_visualization vidoes/  # Original demo videos (converted to GIF for README)
├── README.md                       # Project documentation
```

---

### 10.2 Environment

- Python **3.9+**
- Jupyter Notebook
- Core libraries:
  - `pandas`, `numpy` → data manipulation
  - `scikit-learn` → preprocessing, ML models, metrics
  - `xgboost` → gradient boosting classifier
  - `matplotlib`, `seaborn` → plotting
  - `powerbiclient` → Power BI integration
  - `pickle` → model persistence

Install dependencies:
```bash
pip install -r requirements.txt
```

### 10.3 Dataset Setup

1. **Download CICIDS-2017 dataset** from [UNB official link](https://www.unb.ca/cic/datasets/ids-2017.html).  
2. Place all daily CSV files inside the `Data/` directory of this repo.  
3. The provided notebooks handle the following automatically:
   - **Concatenating** daily CSV files into a unified dataset  
   - **Preprocessing**: dropping IDs/IPs, label encoding, handling NaN/∞ values, scaling features  
   - **Train/Test Split**: ensuring leakage-safe partitioning of data


### 10.4 Running the Project

**Preprocessing + Training**
- Open Jupyter notebooks in **`Python File/`**  
- Run **preprocessing pipeline**  
- Train models: Logistic Regression, Decision Tree, Random Forest, XGBoost, Gaussian Naive Bayes  
- Saved objects appear in **`Packages and Models/`**

---

**Evaluation**
- Run notebooks to generate:  
  - Confusion matrices  
  - Accuracy comparison plots  
  - Feature importance plots  
- Plots auto-save into **`images/`**

---

**PCA Optimization**
- Run PCA variance threshold experiments: **0.90, 0.95, 0.99, 0.999**  
- Compare model performance **with and without PCA**  

---

**Real-Time Simulation**
- Execute **streaming script**  
- Sends predictions to **Power BI REST API**  
- Dashboards update **live**  

---

### 10.5 Power BI Configuration

- Create a **Streaming Dataset** in Power BI  
- Ensure schema matches prediction payload (e.g., `flow_id`, `duration`, `protocol`, `prediction`, `timestamp`)  
- Embed **Power BI API endpoint & key** into your Python script  
- Run streaming script → predictions flow into dashboards **instantly**  

---

### 10.6 Reproducibility Notes

- **Scalers and models** are persisted under **`Packages and Models/`** → reproducible inference without retraining  
- **Random seeds** set in notebooks → consistent splits and metrics  
- **Figures & GIFs** auto-generated → README links always valid  
---

## 11) Reliability & System Design Aspects

While the current system is a **research-to-prototype pipeline**, we explicitly designed for **robustness, scalability, and production readiness**.

---

### 11.1 Reliability Challenges
- **False Negatives (FN)** are the most dangerous → missed attacks.  
  - XGBoost chosen as primary model because it minimizes FN in confusion matrix.  
- **False Positives (FP)** are tolerable but costly → waste analyst time.  
  - Balanced by threshold tuning + ensemble fallback.  
- **Dirty Inputs (NaN, ∞, malformed rows)** → handled in preprocessing to avoid runtime crashes.  
- **Model Drift** → dataset distributions can change; retraining required periodically.

---

### 11.2 Fault Tolerance
- **Model Persistence**: scalers + models saved in `Packages and Models/` → ensures consistent inference even after restarts.  
- **Streaming Resilience**: if Power BI endpoint is unavailable, predictions are buffered until retry.  
- **Reproducibility**: seeds fixed, pipelines documented → results are repeatable.  
- **Logging**: preprocessing pipeline logs dropped/imputed values for auditing.

---

### 11.3 Scaling Considerations
- **Traffic Volume**: Python simulation is sequential; for real-world scale, replace with:
  - Kafka for ingestion
  - Spark Structured Streaming for distributed processing
- **Feature Dimensionality**: PCA reduces latency ~27% at 0.99 variance — critical for large-scale streaming.  
- **Concurrency**: Power BI Streaming Dataset has rate limits (~1 row/sec per tile); can be sharded into multiple datasets for higher throughput.  
- **Parallelism**: scikit-learn + XGBoost can be parallelized with `n_jobs=-1` for multi-core training.

---

### 11.4 Security Considerations
- **Data Leakage**: scaler fit only on train → prevents peeking into test distribution.  
- **PII Sensitivity**: IPs, Flow IDs removed early to prevent storing sensitive identifiers.  
- **Auditability**: dashboards show not just predictions but metrics → builds trust with SOC analysts.

---

### 11.5 Production-Ready Extensions
- **Streaming Upgrade**: Python → Kafka → Spark → ML inference → real-time sink (DB or BI tool).  
- **Alerting Layer**: add email/Slack notifications when attack ratio exceeds thresholds.  
- **Model Registry**: integrate MLflow for versioning, drift monitoring, and rollback.  
- **Deployment**: package as containerized microservice (FastAPI/Flask) with REST endpoints.  
- **Monitoring**: add Prometheus + Grafana for system health metrics.

---

### Takeaway
This project demonstrates **engineering discipline beyond notebooks**:  
- Reliability (robust preprocessing, FN minimization)  
- Fault Tolerance (model persistence, retries, reproducibility)  
- Scalability (PCA, parallelism, potential Spark integration)  
- Security (no leakage, PII-safe preprocessing)  

---

## 12) Skills Showcased & Future Work

---

### 12.1 Skills Demonstrated

** Data Engineering**
- Built a reproducible preprocessing pipeline (drop IDs, handle NaN/∞, leakage-safe scaling).
- Unified multi-day CICIDS-2017 logs into one dataset.
- Managed imbalanced classes and large-scale tabular features.

** Data Analysis**
- Performed exploratory analysis (attack distributions, daily trends, correlation heatmaps).
- Extracted insights linking flow duration, flags, and port usage to malicious behavior.
- Identified redundancy → motivated PCA dimensionality reduction.

** Machine Learning**
- Benchmarked multiple algorithms (LogReg, DT, RF, XGBoost, GNB).
- Deep dive into metrics: Accuracy, Precision, Recall, F1, MCC.
- Interpreted models: feature importance, coefficients, confusion matrices.

** Optimization**
- PCA variance threshold experiments (0.90–0.999).
- Balanced accuracy vs runtime (achieved ~27% faster training with <0.2% accuracy drop).

** Real-Time Simulation**
- Built streaming pipeline: Python → XGBoost inference → Power BI REST API.
- Created **auto-updating dashboards** (KPIs, trends, heatmaps).
- Converted demo videos into GIFs for visual proof.

** System Design Thinking**
- Reliability: minimized false negatives, robust preprocessing.
- Scalability: PCA for latency, parallelism, future Spark integration.
- Fault tolerance: persisted models, retries, reproducibility.
- Security: removed PII, leakage-safe design, audit-friendly dashboards.

** Visualization**
- Static plots: distributions, confusion matrices, feature importance, PCA trade-offs.
- Live dashboards: Power BI streaming datasets with near real-time updates.
- Communicated ML insights in operator-friendly KPIs.

---

### 12.2 Future Extensions

1. **Streaming Upgrade**
   - Replace Python loop with **Kafka ingestion** + **Spark Structured Streaming** for scalability.
   - Handle thousands of flows/sec.

2. **Advanced Modeling**
   - Expand from binary (Benign vs Attack) to **multi-class IDS** (predict attack type).
   - Explore deep learning architectures (LSTMs, Autoencoders) for temporal sequences.

3. **MLOps Integration**
   - Model registry (MLflow) with versioning, A/B testing, and drift monitoring.
   - Automated retraining pipelines.

4. **Alerting & Response**
   - Integrate **Slack/Email/Webhooks** for real-time attack alerts.
   - Add thresholds for triggering alerts (e.g., >10% malicious traffic in last 5 mins).

5. **Security Analytics Integration**
   - Export predictions to a **SIEM system** (Splunk, Azure Sentinel).
   - Join with logs for enterprise-wide monitoring.

6. **Cloud Deployment**
   - Deploy model inference as **REST microservice** (FastAPI).
   - Scale with Kubernetes + autoscaling.

---

### Final Note

This project demonstrates **end-to-end mastery**:
- Data Engineering → ML → Optimization → Visualization → System Design.  
It bridges the gap between **academic ML projects** and **production-grade pipelines**, proving readiness for **data engineering, ML engineering, and security analytics roles**.


