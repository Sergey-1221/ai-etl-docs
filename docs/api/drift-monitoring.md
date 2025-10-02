# 📊 Drift Monitoring API

## Overview

The Drift Monitoring API detects data drift, feature drift, and target drift in ML pipelines using Evidently AI. Automatically triggers retraining when drift is detected.

## Base Endpoints

All drift monitoring endpoints are under `/api/v1/drift-monitoring`

## Authentication

```http
Authorization: Bearer {your_access_token}
```

---

## Data Drift Detection

### Detect Data Drift

Compare reference and current datasets to detect drift.

```http
POST /api/v1/drift-monitoring/detect-drift
Content-Type: application/json
Authorization: Bearer {token}

{
  "reference_data": {
    "source": "s3://data-lake/reference/customer_data_2024_01.parquet",
    "date_range": {
      "start": "2024-01-01",
      "end": "2024-01-31"
    }
  },
  "current_data": {
    "source": "s3://data-lake/current/customer_data_2024_06.parquet",
    "date_range": {
      "start": "2024-06-01",
      "end": "2024-06-30"
    }
  },
  "drift_config": {
    "threshold": 0.1,
    "stattest": "psi",
    "columns": ["age", "income", "purchase_count", "days_since_signup"]
  }
}
```

**Parameters:**
- `reference_data`: Baseline dataset (training data)
- `current_data`: Production data to compare
- `drift_config.threshold`: Drift detection threshold (0.0-1.0)
- `drift_config.stattest`: Statistical test ("psi", "ks", "wasserstein", "jensenshannon")
- `drift_config.columns`: Columns to monitor (optional, defaults to all numeric)

**Response:**
```json
{
  "drift_detected": true,
  "drift_score": 0.24,
  "drift_share": 0.45,
  "columns_with_drift": [
    {
      "column": "income",
      "drift_score": 0.35,
      "stattest": "psi",
      "threshold": 0.1,
      "drift_detected": true,
      "distribution_change": {
        "reference_mean": 65000,
        "current_mean": 72000,
        "percentage_change": 10.77
      }
    },
    {
      "column": "purchase_count",
      "drift_score": 0.28,
      "stattest": "psi",
      "threshold": 0.1,
      "drift_detected": true,
      "distribution_change": {
        "reference_mean": 3.2,
        "current_mean": 2.1,
        "percentage_change": -34.38
      }
    }
  ],
  "report_url": "https://storage.ai-etl.com/reports/drift_2024_06_30.html",
  "recommendation": "RETRAIN_REQUIRED",
  "confidence": 0.89
}
```

---

## Feature Drift Analysis

### Analyze Feature-Level Drift

Detailed drift analysis for each feature with statistical tests.

```http
POST /api/v1/drift-monitoring/feature-drift
Content-Type: application/json
Authorization: Bearer {token}

{
  "pipeline_id": "pipe_customer_churn",
  "reference_period": {
    "start": "2024-01-01",
    "end": "2024-03-31"
  },
  "current_period": {
    "start": "2024-06-01",
    "end": "2024-06-30"
  },
  "features": [
    "age",
    "income",
    "tenure_days",
    "total_purchases",
    "avg_order_value",
    "support_tickets_count"
  ],
  "target_column": "churned",
  "stattest_config": {
    "numerical": "ks",
    "categorical": "chisquare"
  }
}
```

**Response:**
```json
{
  "pipeline_id": "pipe_customer_churn",
  "analysis_timestamp": "2024-06-30T10:00:00Z",
  "overall_drift": {
    "detected": true,
    "severity": "high",
    "drift_score": 0.31
  },
  "feature_analysis": [
    {
      "feature": "income",
      "type": "numerical",
      "drift_detected": true,
      "drift_score": 0.42,
      "stattest": "ks",
      "p_value": 0.001,
      "statistics": {
        "reference": {
          "mean": 65000,
          "std": 15000,
          "min": 25000,
          "max": 150000,
          "quantiles": {
            "25": 52000,
            "50": 63000,
            "75": 78000
          }
        },
        "current": {
          "mean": 72000,
          "std": 18000,
          "min": 28000,
          "max": 180000,
          "quantiles": {
            "25": 58000,
            "50": 71000,
            "75": 86000
          }
        }
      },
      "visualization_url": "https://storage.ai-etl.com/viz/income_drift.png"
    },
    {
      "feature": "churned",
      "type": "target",
      "drift_detected": true,
      "drift_score": 0.28,
      "target_drift_type": "label_shift",
      "statistics": {
        "reference_rate": 0.15,
        "current_rate": 0.22,
        "change_percentage": 46.67
      }
    }
  ],
  "recommendations": [
    {
      "action": "RETRAIN_MODEL",
      "priority": "high",
      "reason": "Significant drift detected in 3 out of 6 features",
      "estimated_impact": "Model accuracy likely degraded by 10-15%"
    },
    {
      "action": "UPDATE_PREPROCESSING",
      "priority": "medium",
      "reason": "Income distribution shifted - consider updating normalization"
    }
  ],
  "report_html": "https://storage.ai-etl.com/reports/feature_drift_2024_06_30.html"
}
```

---

## Drift Summary Report

### Get Comprehensive Drift Summary

Generate complete drift report with visualizations.

```http
POST /api/v1/drift-monitoring/drift-summary
Content-Type: application/json
Authorization: Bearer {token}

{
  "pipeline_id": "pipe_customer_churn",
  "time_window": "last_30_days",
  "include_visualizations": true,
  "include_recommendations": true
}
```

**Response:**
```json
{
  "pipeline_id": "pipe_customer_churn",
  "summary": {
    "total_features_monitored": 15,
    "features_with_drift": 5,
    "drift_percentage": 33.33,
    "overall_severity": "medium",
    "drift_trend": "increasing"
  },
  "drift_timeline": [
    {
      "date": "2024-06-01",
      "drift_score": 0.12,
      "features_drifted": 2
    },
    {
      "date": "2024-06-15",
      "drift_score": 0.18,
      "features_drifted": 3
    },
    {
      "date": "2024-06-30",
      "drift_score": 0.31,
      "features_drifted": 5
    }
  ],
  "model_performance_impact": {
    "predicted_accuracy_drop": 12.5,
    "confidence_interval": [10.2, 14.8],
    "recommendation": "Immediate retraining recommended"
  },
  "visualizations": {
    "drift_heatmap": "https://storage.ai-etl.com/viz/drift_heatmap.png",
    "feature_importance_vs_drift": "https://storage.ai-etl.com/viz/importance_drift.png",
    "distribution_comparison": "https://storage.ai-etl.com/viz/distributions.png"
  },
  "html_report_url": "https://storage.ai-etl.com/reports/drift_summary_2024_06_30.html"
}
```

---

## File Upload and Detection

### Upload Files and Detect Drift

Upload reference and current datasets directly.

```http
POST /api/v1/drift-monitoring/upload-drift-detection
Content-Type: multipart/form-data
Authorization: Bearer {token}

Form Data:
- reference_file: reference_data.csv
- current_file: current_data.csv
- threshold: 0.1
- stattest: psi
- target_column: churned (optional)
```

**Response:**
```json
{
  "upload_status": "success",
  "reference_rows": 50000,
  "current_rows": 12000,
  "drift_analysis": {
    "drift_detected": true,
    "drift_score": 0.27,
    "columns_analyzed": 12,
    "columns_with_drift": 4
  },
  "report_url": "https://storage.ai-etl.com/reports/upload_drift_12345.html"
}
```

---

## Retraining Management

### Check Retraining Status

Check if model retraining is recommended.

```http
GET /api/v1/drift-monitoring/retraining-status?pipeline_id=pipe_customer_churn
Authorization: Bearer {token}
```

**Response:**
```json
{
  "pipeline_id": "pipe_customer_churn",
  "retraining_required": true,
  "last_training_date": "2024-03-15T08:00:00Z",
  "days_since_training": 107,
  "drift_severity": "high",
  "auto_retrain_scheduled": true,
  "scheduled_retrain_date": "2024-07-01T02:00:00Z",
  "trigger_conditions": [
    {
      "condition": "drift_score > 0.25",
      "current_value": 0.31,
      "threshold": 0.25,
      "status": "triggered"
    },
    {
      "condition": "drift_features > 3",
      "current_value": 5,
      "threshold": 3,
      "status": "triggered"
    }
  ]
}
```

---

## Configuration

### Get Drift Monitoring Configuration

```http
GET /api/v1/drift-monitoring/config
Authorization: Bearer {token}
```

**Response:**
```json
{
  "default_threshold": 0.1,
  "default_stattest": {
    "numerical": "psi",
    "categorical": "chisquare"
  },
  "auto_retrain_enabled": true,
  "retraining_triggers": {
    "drift_score_threshold": 0.25,
    "min_features_drifted": 3,
    "days_since_training": 90
  },
  "monitoring_frequency": "daily",
  "alert_channels": ["email", "slack"],
  "retention_days": 90
}
```

---

## Use Cases

### 1. Continuous ML Monitoring

```python
import httpx

async def monitor_ml_pipeline(pipeline_id: str):
    """Monitor ML pipeline for drift continuously"""

    # Detect drift
    response = await client.post("/drift-monitoring/feature-drift", json={
        "pipeline_id": pipeline_id,
        "reference_period": {"start": "2024-01-01", "end": "2024-03-31"},
        "current_period": {"start": "2024-06-01", "end": "2024-06-30"},
        "features": ["age", "income", "tenure_days", "purchases"],
        "target_column": "churned"
    })

    result = response.json()

    if result["overall_drift"]["detected"]:
        # Check if retraining is needed
        retrain_status = await client.get(
            f"/drift-monitoring/retraining-status?pipeline_id={pipeline_id}"
        )

        if retrain_status.json()["retraining_required"]:
            # Trigger retraining
            await trigger_model_retraining(pipeline_id)

    return result
```

### 2. Automated Drift Alerts

```python
async def setup_drift_alerts(pipeline_id: str, email: str):
    """Set up automated drift detection alerts"""

    # Configure monitoring
    config = {
        "pipeline_id": pipeline_id,
        "monitoring_frequency": "daily",
        "alert_threshold": 0.15,
        "alert_channels": {
            "email": email,
            "slack_webhook": "https://hooks.slack.com/..."
        },
        "auto_retrain": True,
        "retrain_threshold": 0.25
    }

    await client.post("/drift-monitoring/configure-alerts", json=config)
```

### 3. Batch Drift Detection

```python
async def batch_drift_detection(pipelines: list):
    """Detect drift for multiple pipelines"""

    results = []

    for pipeline_id in pipelines:
        summary = await client.post("/drift-monitoring/drift-summary", json={
            "pipeline_id": pipeline_id,
            "time_window": "last_7_days"
        })

        results.append({
            "pipeline_id": pipeline_id,
            "drift_detected": summary.json()["summary"]["features_with_drift"] > 0
        })

    return results
```

---

## Statistical Tests

### Available Tests

| Test | Type | Best For | Interpretation |
|------|------|----------|----------------|
| **PSI** (Population Stability Index) | Numerical | Distribution changes | > 0.1: drift, > 0.25: severe |
| **KS** (Kolmogorov-Smirnov) | Numerical | Distribution differences | p-value < 0.05: drift |
| **Wasserstein** | Numerical | Distance between distributions | Higher = more drift |
| **Jensen-Shannon** | Numerical | Symmetric divergence | 0-1 scale, > 0.1: drift |
| **Chi-Square** | Categorical | Category distribution | p-value < 0.05: drift |
| **Z-test** | Target | Target rate changes | p-value < 0.05: drift |

---

## Performance

### Detection Time

| Dataset Size | Features | Detection Time | Memory Usage |
|--------------|----------|----------------|--------------|
| 10K rows | 10 | 2-3s | 50MB |
| 100K rows | 20 | 8-12s | 200MB |
| 1M rows | 50 | 45-60s | 1.5GB |

### Best Practices

1. **Reference Data**: Use training data or stable production period
2. **Sample Size**: Minimum 1000 rows for reliable statistics
3. **Update Frequency**: Daily for critical models, weekly for stable ones
4. **Threshold Tuning**: Start with 0.1-0.15, adjust based on false positives

---

## Error Handling

### Common Errors

```json
{
  "error": {
    "code": "INSUFFICIENT_DATA",
    "message": "Reference dataset has only 150 rows, minimum 1000 required",
    "details": {
      "reference_rows": 150,
      "minimum_required": 1000
    }
  }
}
```

---

## Related Documentation

- [Drift Monitoring Service](../services/drift-monitoring-service.md)
- [ML Pipeline Optimization](../guides/ml-optimization.md)
- [Evidently AI Integration](../AI_ENHANCEMENTS_SETUP.md)

---

[← Back to API Reference](./rest-api.md)
