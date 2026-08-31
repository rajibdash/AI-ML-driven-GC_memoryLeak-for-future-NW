# Architecture Design Document: AI-Driven Root Cause Analysis for C++ Memory Leaks
### Author: Rajib Kumar Dash

## 1. Executive Summary & Core Design
The goal of this system is to deploy an automated, closed-loop engineering pipeline that detects, triages, and fixes memory leaks in large-scale C++ applications. 
By combining structural code analysis with Machine Learning models and Generative AI, the platform acts as an autonomous site reliability and security engineer.


## 2. Document Overview
This document specifies the design, algorithmic framework, machine learning architecture, and operational roadmap for an automated system capable of detecting, diagnosing, and patching memory leaks in modern C++ codebases.

```
+-----------------------------------------------------------------------------------+
|                              1. Telemetry(or Data) Ingestion Layer                         |
|   [ASan / LSan Raw Stderr Logs] -> [Symbolication Engine (addr2line / c++filt)]   |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
|                            2. Context Enrichment Layer                            |
|    [LibClang AST Extractor] + [Git Blame Meta-Parser] + [Control Flow Generator]  |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
|                        3. Triage & Machine Learning Layer                         |
|   [Feature Vector Generator] -> [Random Forest / GNN Triage Model] -> Profile ID  |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
|                         4. LLM Patch & Verification Loop                          |
|    [Multi-Agent Prompt Chain] -> [Isolated Sandboxed Patch Engine (CMake/Bazel)]  |
+-----------------------------------------------------------------------------------+
```

---

## 3. Machine Learning Architecture & Comprehensive Model Training

Before passing code to an expensive LLM context window, a specialized Machine Learning model triages the memory leak profile. This stage accurately categorizes leaks, enabling the selection of highly specific, localized prompt templates and context strategies.

### 3.1 Feature Engineering Pipeline
Raw text traces and AST data are transformed into a multi-dimensional mathematical vector space:
* **Trace Depth (`trace_depth`):** Integer count of the call stack frames. Shallow traces often indicate direct allocation omissions, while deep traces point to architectural mismatches or callback registrations.
* **Allocation Scope (`alloc_in_loop`):** Binary flag indicating if the allocation occurred inside a loop context (`for`, `while`, `do-while`).
* **Smart Pointer Context (`smart_ptr_density`):** Floating-point index representing the ratio of modern smart pointer keywords (`std::unique_ptr`, `std::shared_ptr`, `std::make_unique`) to raw pointer variables inside the containing source module.
* **Cyclic Tendency (`nested_class_depth`):** Numerical depth of classes/structs containing structural declarations pointing back to parent contexts or peer instances.
* **Error/Exception Pathway (`try_catch_presence`):** Binary representation indicating if an allocation sits between a `try` block start and its matching `catch` boundary or early-return path.

### 3.2 Deep Dive: Comprehensive Model Training Lifecycle

To build a production-grade predictive triage system, the pipeline must follow a deterministic training infrastructure.

```
+-------------------+      +-------------------+      +-------------------+
| 1. Synthetic Leak | ---> | 2. Data Cleaning  | ---> | 3. Class Balance  |
|    Generation     |      |  & Normalization  |      |   via SMOTE-NC    |
+-------------------+      +-------------------+      +-------------------+
                                                                |
                                                                v
+-------------------+      +-------------------+      +-------------------+
| 6. Deployment     | <--- | 5. Hyperparameter | <--- | 4. Cross-Val &    |
|    & Shadowing    |      |    Optimization   |      |  Evaluation Loop  |
+-------------------+      +-------------------+      +-------------------+
```

#### Step 1: Data Collection & Synthetic Leak Generation
Because production datasets are heavily skewed towards resolved historical bugs, we generate synthetic training data:
* **Clang-Based Chaos Testing:** A custom parser automatically injects diverse leaks into open-source modern C++ repositories (e.g., stripping `delete` statements, wrapping allocations in complex exception branches, or overriding class destructors to be non-virtual).
* **Automated Profiling Suite:** These altered codebases are executed under continuous integration test harnesses with `ASan`/`LSan` configurations to yield rich, multi-layered ground-truth profiles.

#### Step 2: Data Cleaning, Preprocessing & Feature Scaling
Raw features from logs can vary greatly in range (e.g., file line numbers vs. smart pointer density percentages).
* **Missing Value Imputation:** Optional feature markers (such as Git lifetime attributes) are filled dynamically using regional medians if the commit history is truncated.
* **Standard Scaling:** Features such as stack depths are normalized using a standard scaling method ($z = (x - \mu) / \sigma$) to maintain variance boundaries across linear and distance-based scoring systems.

#### Step 3: Resolving Class Imbalance via SMOTE-NC
In practical software systems, some leak types occur much more frequently than others (e.g., forgotten deletes are common, while multi-threaded cyclic resource leaks are rare but critical).
* **Oversampling Execution:** The pipeline implements Synthetic Minority Over-sampling Technique for Nominal and Continuous features (**SMOTE-NC**). This technique creates realistic synthetic minority data points without duplicating records, preventing the classifier from developing biases toward high-frequency patterns.

#### Step 4: Stratified Cross-Validation & Metric Evaluation
Models are evaluated using **5-Fold Stratified Cross-Validation** to verify their stability across varying file types and modules.
* **Accuracy:** Overall correctness score across all triage classifications.
* **Precision / Macro F1-Score:** Crucial for optimization; misclassifying a complex multi-threaded structural leak as a basic forgotten delete causes the system to generate incorrect patches, wasting sandbox validation compute cycles.

#### Step 5: Hyperparameter Optimization
An automated grid optimization strategy searches through hyperparameter combinations to minimize overfitting:
* **Estimator Volume (`n_estimators`):** Checked across ranges `[50, 100, 200, 500]`.
* **Tree Structure Limits (`max_depth`):** Constrained to levels `[5, 10, 20, None]` to prevent memorization of specific class or file naming conventions.

#### Step 6: Production Deployment & Shadowing
Once validation metrics stabilize above a **95% F1-Score Threshold**, the model is deployed into a *Shadow Mode* pipeline. It analyzes incoming pull requests and logs predictions alongside human developer reviews to track drift prior to activating automated patch generation.

---

### 3.3 Executable Training Pipeline Implementation

The script below shows how to train, optimize, balance, evaluate, and save the core leak-triage model using a comprehensive framework.

```python
import numpy as np
import pandas as pd
import joblib
from sklearn.model_selection import train_test_split, StratifiedKFold, GridSearchCV
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import classification_report, confusion_matrix, accuracy_score

# ==========================================
# 1. SYNTHETIC DATA GENERATION (COMPREHENSIVE LABELED SET)
# ==========================================
def generate_comprehensive_leak_dataset(samples=2000):
    np.random.seed(42)
    
    # Class Profiles:
    # 0: ForgottenDelete (High frequency)
    # 1: ExceptionPathBypass (Medium frequency)
    # 2: CyclicReference (Low frequency - minority class)
    
    # Features: [trace_depth, alloc_in_loop, smart_ptr_density, nested_class_depth, try_catch_presence]
    data = []
    labels = []
    
    for _ in range(samples):
        class_choice = np.random.choice([0, 1, 2], p=[0.60, 0.30, 0.10])
        
        if class_choice == 0:
            # ForgottenDelete: Shallow stack, often in loop, low smart pointer usage
            trace_depth = np.random.randint(2, 8)
            alloc_in_loop = np.random.choice([0, 1], p=[0.3, 0.7])
            smart_ptr_density = np.random.uniform(0.0, 0.2)
            nested_class_depth = np.random.randint(0, 2)
            try_catch_presence = np.random.choice([0, 1], p=[0.9, 0.1])
            
        elif class_choice == 1:
            # ExceptionPathBypass: Medium stack depth, high try/catch context, erratic returns
            trace_depth = np.random.randint(6, 18)
            alloc_in_loop = np.random.choice([0, 1], p=[0.6, 0.4])
            smart_ptr_density = np.random.uniform(0.1, 0.5)
            nested_class_depth = np.random.randint(0, 3)
            try_catch_presence = np.random.choice([0, 1], p=[0.05, 0.95])
            
        else:
            # CyclicReference: Deep structural layers, high modern pointer usage but misused
            trace_depth = np.random.randint(12, 35)
            alloc_in_loop = np.random.choice([0, 1], p=[0.8, 0.2])
            smart_ptr_density = np.random.uniform(0.6, 1.0)
            nested_class_depth = np.random.randint(3, 7)
            try_catch_presence = np.random.choice([0, 1], p=[0.5, 0.5])
            
        data.append([trace_depth, alloc_in_loop, smart_ptr_density, nested_class_depth, try_catch_presence])
        labels.append(class_choice)
        
    feature_names = ['trace_depth', 'alloc_in_loop', 'smart_ptr_density', 'nested_class_depth', 'try_catch_presence']
    df = pd.DataFrame(data, columns=feature_names)
    df['leak_profile'] = labels
    return df

# Initialize simulation pipeline data
raw_dataset = generate_comprehensive_leak_dataset(samples=2500)
X = raw_dataset.drop(columns=['leak_profile'])
y = raw_dataset['leak_profile']

print(f"[INFO] Initial Class Distribution:
{y.value_counts(normalize=True)}
")

# ==========================================
# 2. DATA SPLITTING & FEATURE PREPROCESSING
# ==========================================
# Stratified split guarantees consistent label ratios between train and validation partitions
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.20, random_state=42, stratify=y
)

# Scale continuous spatial data variables uniformly
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)

# ==========================================
# 3. K-FOLD CROSS-VALIDATION & HYPERPARAMETER TUNING
# ==========================================
print("[INFO] Initiating Grid Search Optimization & Cross-Validation Stratification...")
param_grid = {
    'n_estimators': [100, 200],
    'max_depth': [10, 20, None],
    'min_samples_split': [2, 5],
    'criterion': ['gini', 'entropy']
}

cv_strategy = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
grid_search = GridSearchCV(
    estimator=RandomForestClassifier(random_state=42),
    param_grid=param_grid,
    cv=cv_strategy,
    scoring='f1_macro',
    n_jobs=-1
)

grid_search.fit(X_train_scaled, y_train)
best_model = grid_search.best_estimator_

print(f"[SUCCESS] Optimal Hyperparameters Discovered: {grid_search.best_params_}
")

# ==========================================
# 4. COMPREHENSIVE PERFORMANCE EVALUATION
# ==========================================
y_pred = best_model.predict(X_test_scaled)

accuracy = accuracy_score(y_test, y_pred)
class_report = classification_report(
    y_test, y_pred, target_names=['ForgottenDelete', 'ExceptionPathBypass', 'CyclicReference']
)
conf_matrix = confusion_matrix(y_test, y_pred)

print("==================== MODEL METRICS PERFORMANCE REPORT ====================")
print(f"Overall General Accuracy Index: {accuracy:.4f}
")
print("Detailed Classification Breakdown:")
print(class_report)
print("Confusion Matrix Layout:")
print(conf_matrix)
print("=========================================================================
")

# ==========================================
# 5. SERIALIZATION FOR PRODUCTION ENVIRONMENT
# ==========================================
model_filename = 'cpp_leak_triage_model.pkl'
scaler_filename = 'cpp_leak_scaler.pkl'

joblib.dump(best_model, model_filename)
joblib.dump(scaler, scaler_filename)
print(f"[DEPLOY] Model saved to '{model_filename}' and scaler to '{scaler_filename}'.")
```

---

## 4. Way Forward, Future Improvements & Mathematical Verification

To progress beyond heuristic models, the platform introduces three advanced system layers:

### 4.1 Code Property Graphs (CPGs) & Graph Neural Networks (GNNs)
Flattening source metadata into simple numerical rows ignores structural graph connections. Next-generation implementations parse source files into **Code Property Graphs (CPGs)**, which combine Abstract Syntax Trees (AST), Control Flow Graphs (CFG), and Program Dependence Graphs (PDG) into a single unified representation.
* **GNN Classification:** Graph Neural Networks use vector message-passing along structural code connections. This detects deep structural memory leaks (e.g., aliased references or custom allocators) across split modules that linear models cannot identify.

### 4.2 Differential Verification Testing (DVT)
To ensure automated code changes do not alter performance profiles, the platform runs side-by-side **Differential Verification Testing (DVT)**:
$$T_{	ext{evaluation}} = \mathcal{M}_{	ext{patched}}(I) \equiv \mathcal{M}_{	ext{baseline}}(I)$$
* The original and patched binary versions are run concurrently in separate sandboxes under heavy, randomized load test sequences ($I$).
* **Differential Checking:** The system monitors and compares CPU execution paths, heap limits, cache efficiency, and execution times. A patch is rolled back if it introduces performance degradation or alters deterministic application logic.

### 4.3 Satisfiability Modulo Theories (SMT) Solver Verification
When integrating automated locking mechanisms or modern lifetime wrappers (such as `std::shared_ptr`), the system creates mathematical logic proofs for safety:
* **Formal Solvers:** The pipeline converts structural resource flows into logical equations verified by systems like **Z3 Solver**.
* This mathematically guarantees that an automated leak fix will not create circular wait states, deadlocks, or thread-safety regression paths on concurrent execution boundaries.

---

## 5. Engineering Roadmap

```
+-----------------------------------------------------------------------------------+
|  PHASE 1: Core Automation Foundations (Months 1 - 4)                              |
|  - Stand up LSan/ASan CI automation collectors.                                   |
|  - Implement Clang AST structural parsing.                                        |
|  - Deploy the production triage classifier script in shadow mode.                 |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
|  PHASE 2: Agentic Repair & Verification Loops (Months 5 - 8)                      |
|  - Integrate LLM multi-agent prompt loops.                                        |
|  - Build isolated compilation sandboxes (CMake/Bazel).                            |
|  - Automate verified Pull Request creation for standard leak categories.          |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
|  PHASE 3: Autonomous Scale & Advanced Logic Models (Months 9 - 12+)               |
|  - Replace linear classifiers with GNN Graph Property models.                     |
|  - Activate continuous execution Differential Verification Testing (DVT).         |
|  - Deploy SMT solvers to verify thread safety bounds automatically.               |
+-----------------------------------------------------------------------------------+
```