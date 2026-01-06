In a Data Mesh, you often need to share data with a "Consumer" project without giving them access to your raw "Producer" project.

The **Authorized View** is the gold standard for this. It allows you to share a SQL query result (the data product) while keeping the underlying source tables completely hidden and protected.

---

## 1. The Cross-Project Architecture

To implement this, you create a "Sharing Dataset" in the Producer project. This dataset contains only the views you want to expose.

### Why this works:

* **Zero Data Movement:** The consumer queries the view, and BigQuery handles the rest. No data is copied.
* **Security:** Even if the consumer has `BigQuery Data Viewer` access to the View, they **cannot** see the source tables. BigQuery uses the "Identity" of the view itself to fetch data.
* **Abstraction:** You can change your internal table schema (e.g., rename columns in `raw_table`) and just update the View's SQL. The consumer's experience remains identical.

---

## 2. Terraform Implementation

This is a two-step process: you must create the view and then **authorize** that view to talk to the source dataset.

### Step A: Create the View

```hcl
# The "Public" face of your data product
resource "google_bigquery_table" "marketing_public_view" {
  project    = "marketing-producer-prod"
  dataset_id = "shared_marketing_data" # A separate dataset for sharing
  table_id   = "customer_summary_v1"

  view {
    query = <<EOF
      SELECT customer_id, region, total_spend 
      FROM `marketing-producer-prod.internal_raw_data.customers`
      WHERE is_active = true
    EOF
    use_legacy_sql = false
  }
}

```

### Step B: Authorize the View

You must tell the **Internal Dataset** that it is allowed to be read by the **View** in the Shared Dataset.

```hcl
resource "google_bigquery_dataset_access" "authorize_marketing_view" {
  project    = "marketing-producer-prod"
  dataset_id = "internal_raw_data" # The source dataset

  view {
    project_id = google_bigquery_table.marketing_public_view.project
    dataset_id = google_bigquery_table.marketing_public_view.dataset_id
    table_id   = google_bigquery_table.marketing_public_view.table_id
  }
}

```

---

## 3. When to use Analytics Hub vs. Authorized Views

As your Data Mesh grows, you might find managing hundreds of individual view authorizations difficult.

| Feature | Authorized Views | BigQuery Analytics Hub |
| --- | --- | --- |
| **Best For** | Internal sharing between 2-3 projects. | Scalable data marketplace for the whole org. |
| **User Experience** | Hand-managed IAM roles. | "Browse and Subscribe" UI (Self-service). |
| **Discovery** | Hard to find without documentation. | Featured in a central Data Exchange. |
| **Granularity** | Table/View level. | Dataset level (can share many views at once). |

---

### Final Design Tip

In a mature Data Mesh, use **Authorized Datasets**. This allows you to authorize an *entire dataset* of views at once. Any new view you add to that "Shared" dataset automatically gets permission to read from your "Internal" source.

**Would you like me to provide the IAM roles required for a "Consumer" service account to query these views from a different project?**