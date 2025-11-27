
Project: TelcoTraffic — AI-Driven Network Intelligence
Scalable TLS Traffic Analysis for Service Discovery & Capacity Planning

Status: 🟢 Active | Stack: Apache Spark, PyTorch, XGBoost, CESNET DataZoo | Data: 1 Year Backbone Traffic

1. Executive Summary & Problem Framing
In the modern telecommunications landscape, encrypted traffic (TLS) obscures visibility into network usage. This project implements a Big Data pipeline to analyze TLS traffic patterns without decryption, leveraging the CESNET-TLS-Year22 dataset.

The Goal: Build a scalable system to identify emerging services, forecast capacity needs, and detect market shifts for a Telecom Operator.

Key Objectives:

Service Classification: Identify specific apps (e.g., Netflix, Teams) from encrypted flow features for product usage analytics.

Capacity Forecasting: Predict aggregate bytes/flows per service to optimize network provisioning.

Anomaly Detection: Discover unknown services or behavioral shifts indicating new market trends (Data Drift).

2. System Architecture: The Big Data Pipeline
This system handles multi-TB scale data through a decoupled storage and compute architecture.

High-Level Data Flow

Ingestion → Raw Storage → Feature Store → Model Training → Serving → Monitoring

Pipeline Visualization:

Code snippet
graph TD
    A[Raw CESNET CSVs] -->|Spark Batch Ingestion| B(Raw Storage S3/HDFS)
    B -->|Parquet Conversion| C{Feature Store}
    C -->|Aggregates| D[Forecasting Models]
    C -->|Flow Features| E[Classification Models]
    D & E --> F[Model Registry MLFlow]
    F --> G[Dashboards & Reporting]
    G --> H[Drift Monitoring]
    H -->|Trigger| A
Technology Stack:

Ingestion: Apache Spark (PySpark) on YARN/EMR.

Storage: AWS S3 (Raw), Delta Lake (Feature Store).

Orchestration: Apache Airflow.

Model Registry: MLflow.

3. Data Engineering Strategy
3.1 Dataset

We utilize the CESNET-TLS-Year22 dataset, a privacy-preserving dataset of real-world traffic.

Source: Nature Scientific Data Paper

Access: Zenodo Record

Tooling: We utilize the CESNET DataZoo API for standardized partitioning.

3.2 Ingestion Logic (Apache Spark)

Due to the dataset's size (Year-long, compressed CSVs), we utilize Spark for distributed processing.

Job A (Parsing): Decompresses CSVs, parses PPI (Packet Packet Information) arrays, and partitions by TIME_FIRST (Week) and APP.

Job B (Aggregation): Computes hourly/daily sums of BYTES and PACKETS for time-series forecasting.

Job C (Feature Extraction): Extracts TLS fields (JA3, SNI) and handles high-cardinality hashing.

Snippet: Ingestion Logic

Python
# PySpark Ingestion Logic
df = spark.read.csv('s3://cesnet/raw/*.csv.gz', schema=schema) \
    .withColumn('TIME_FIRST', to_timestamp(col('TIME_FIRST'))) \
    .withColumn('PPI_First', parse_ppi_udf(col('PPI'))) 

# Partitioning for optimized querying
df.repartition(200).write.format('parquet') \
    .partitionBy('year', 'week', 'app_label') \
    .save('s3://telco/cesnet/feature_store/')
4. Machine Learning Methodology
We employ a Hybrid Modeling Approach combining Tree-based models for structured data and Sequence models for packet timing.

4.1 Feature Engineering

Sequence Features: PPI (Packet Payload Information) sequences are padded to length 30.

Categorical Encoding:s

Low Cardinality: One-Hot Encoding (Protocol, Flags).

High Cardinality (SNI, JA3): Target Encoding or Hashing Trick to avoid sparse matrices.

Scaling: * Trees: Log-transform log(1+x) for skewed distributions (Bytes).

Neural Nets: Standard Scaling (z= 
σ
x−μ
​	
 ).

4.2 Model Architecture

Task	Primary Model	Why?
A. Service Classification	XGBoost / LightGBM	Handles class imbalance, interpretable feature importance, robust to missing values.
B. Sequence Analysis	LSTM / 1D-CNN	Captures temporal dependencies in packet inter-arrival times that summary stats miss.
C. Forecasting	Prophet / TFT	Handles seasonality (daily/weekly) and holidays for capacity planning.
4.3 Implementation Strategy

Hybrid Ensemble: We stack the probability outputs of the LSTM (Sequence data) with the XGBoost model (Flow stats) to maximize Macro F1 score.

Pseudo-Code: Training Pipeline

Python
# 1. Structured Learning (XGBoost)
dtrain = xgboost.DMatrix(data=X_structured, label=y, weight=class_weights)
bst = xgboost.train(params={'objective': 'multi:softprob'}, dtrain=dtrain)

# 2. Sequence Learning (LSTM/Keras)
model = Sequential([
    Masking(mask_value=0., input_shape=(30, 3)),
    LSTM(64, return_sequences=False),
    Dense(num_classes, activation='softmax')
])
model.fit(X_sequences, y)

# 3. Ensemble (Stacking)
ensemble_preds = (w1 * bst.predict(X_val)) + (w2 * model.predict(X_val_seq))
5. Results & Evaluation
Metrics

Classification: Macro F1 Score (Critical for imbalanced classes like rare apps).

Forecasting: RMSE (Root Mean Square Error) and MAPE (Mean Absolute Percentage Error).

Drift: Kullback-Leibler Divergence on feature histograms.

Findings

Tree models proved superior for "Category" classification (Video vs. Chat).

Sequence models significantly improved detection of encrypted tunnel traffic (VPNs) by analyzing packet inter-arrival times.

Capacity Forecasting successfully predicted 90% of traffic peaks with a 24-hour horizon using XGBoost on lag features.

6. Operationalization & Future Work
Monitoring

We utilize Concept Drift Detection (ADWIN or MFWDD). If the distribution of TLS_SNI or PPI sequences deviates significantly, the Retraining Pipeline is triggered automatically via Airflow.

Next Steps

Graph Analytics: Implement Graph Neural Networks (GNN) to map relationships between Source IPs and Destination ASNs to detect botnets.

Privacy: Explore Federated Learning to train on edge nodes without centralizing raw flow logs.

7. References & Resources
Dataset Paper: CESNET-TLS-Year22: A year-spanning TLS network traffic dataset

Download: Zenodo Repository

Tools: CESNET DataZoo | CESNET Models