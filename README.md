# NSUAD Anomaly Detection Demo

This repository contains a reproducible implementation and demonstration of **NSUAD**, a deep-learning–based model for network anomaly detection.  
The project includes a complete environment setup (Python 3.6 + TensorFlow 1.10), a deterministic training pipeline,  
and a runnable **demo notebook** that walks through dataset preparation, model training, and evaluation metrics.

---

## 🚀 Overview

NSUAD is a weakly-supervised deviation-network–based anomaly detector.  
It trains a neural model to produce **deviation scores**, which separate normal samples (inliers) from anomalous samples (outliers).

This repository demonstrates:

- preprocessing data  
- building the DEVNet architecture  
- training with deterministic seeding  
- computing ROC-AUC and AP metrics  
- visualizing score distributions  
- moving a portion of anomalies from the test set into the train set  
- plotting histograms and diagnostic charts  

The full workflow is implemented in **`demo.ipynb`**.

---

## 📦 Structure

/
├── demo.ipynb          # Main entrypoint demo notebook
├── README.md           # This file
├── project_info.md     # Maintainer + metadata
└── 

---

## ▶️ Running the Demo

### **1. Create the required Python environment**

DEVNet requires **Python 3.6.6**

### **2. Install dependencies**

```bash
pip install tensorflow==1.10.0
pip install keras==2.2.4
pip install numpy==1.14.5
pip install pandas==0.23.4
pip install scikit-learn==0.20.0
pip install matplotlib
```

### **3. Launch the demo**

```bash
jupyter notebook demo.ipynb
```


---

## 📊 What the Demo Produces

The demo walks through:

- deterministic seeding for reproducible runs  
- splitting anomalies between train/test  
- training a deviation network  
- computing evaluation metrics  
- plotting score distributions  
- visualizing model behavior  

Expected outputs include:

- printed **AUC-ROC** and **AUC-PR** values  
- histograms showing inlier/outlier score separation  
- diagnostic runtime logs  


Original Paper:
- Johari SS, Shahriar N, Tornatore M, Boutaba R, Saleh A. Anomaly detection and localization in nfv systems: an unsupervised learning approach. In2022 IEEE/IFIP Network Operations and Management Symposium, NOMS 2022 2022 (pp. 1-9). IEEE.
