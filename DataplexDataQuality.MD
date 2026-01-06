In Dataplex, you define **Data Quality (DQ) scans** to validate your data products. Instead of writing complex SQL for every table, you can use a YAML-based specification to define rules like "this column cannot be null" or "this value must be within a specific range."

---

### 1. The YAML Rule Specification

You can store this YAML file in a Cloud Storage bucket. It acts as the "contract" for your data product's reliability.

```yaml
# marketing_dq_rules.yaml
rules:
  - column: customer_id
    dimension: VALIDITY
    threshold: 1.0  # 100% of rows must pass
    non_null_expectation: {}
  - column: lead_score
    dimension: VALIDITY
    threshold: 0.95 # 95% must pass
    range_expectation:
      min_value: 0
      max_value: 100
  - column: signup_date
    dimension: FRESHNESS
    threshold: 1.0
    table_row_count_expectation:
      minimal_row_count: 1

```

---

### 2. Automating the Scan with Terraform

To run these rules automatically, you create a `google_dataplex_datascan`. This resource points to your BigQuery table and the YAML rules defined above.

```hcl
resource "google_dataplex_datascan" "marketing_dq_scan" {
  data_scan_id = "marketing-customer-dq-scan"
  location     = "us-central1"
  project      = "your-governance-project-id"

  # The data product being scanned
  data {
    resource = "//bigquery.googleapis.com/projects/marketing-prod/datasets/customer_data/tables/profiles"
  }

  # The DQ specification
  data_quality_spec {
    sampling_percent = 100
    row_filter       = "signup_date > '2023-01-01'" # Scan only recent data
    
    # You can define rules directly in HCL or link to the YAML file
    rules {
      column = "email"
      dimension = "COMPLEATNESS"
      non_null_expectation {}
    }
  }

  execution_spec {
    trigger {
      schedule {
        cron = "0 8 * * *" # Run every day at 8 AM
      }
    }
  }
}

```

---

### 3. Monitoring Data Health

Once the scan runs, Dataplex pushes the results to the **Data Catalog**.

* **Metadata Integration:** When an analyst searches for the `profiles` table in the Data Catalog, they will see a "Data Quality" tab showing whether the last scan passed or failed.
* **Alerting:** You can route these scan results to **Cloud Logging** and set up an alert to notify the Marketing team via Slack or PagerDuty if their data product "breaks" its quality contract.

---

### Why this is critical for Data Mesh

In a decentralized mesh, the central team cannot manually check everyone's data. By providing this **Self-Serve Platform** tool:

1. **Producers** (Marketing) take responsibility for their own quality scores.
2. **Consumers** (Finance) gain trust in the data because they can see the "Quality Stamp" before using it.

**Would you like to see how to set up an "Authorized View" in BigQuery to safely share these data products across projects without moving the data?**