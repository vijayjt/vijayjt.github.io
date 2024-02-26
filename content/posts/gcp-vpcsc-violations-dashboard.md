+++ 
draft = false
date = 2024-02-25T08:10:04Z
title = "GCP VPC Service Controls Dashboard"
description = ""
slug = ""
authors = []
tags = ["google","cloud", "gcp", "vpcsc", "bigquery", "looker"]
categories = []
externalLink = ""
series = []
+++

[Lorenzo Caggioni](https://medium.com/@lcaggio) has a great blog [post](https://medium.com/google-cloud/create-a-data-studio-dashboard-to-monitor-vpc-sc-violations-on-your-google-cloud-organization-bf8f3bead691) on how to visualise VPC Service Controls violations using Big Query and Looker Studio (formerly Data Studio). The dashboard is super useful for understanding which services, identities or GCP projects are generating the most violations. This makes it easier to determine where to start focusing your efforts in addressing the violations.

This short post provides the terraform code to:

* Create the log sink to Big Query for VPC SC.
* Create a table to use for the dashboard.

```Terraform
variable "gcp_org_id" {
  type        = string
  description = "The GCP Organization ID"
  default     = "changme"
}

locals { 
  bq_project = "changeme"
  org_service_account = "service-org-${var.gcp_org_id}@gcp-sa-logging.iam.gserviceaccount.com"
  # You can customise this filter to only show logs for specific services
  base_filter = <<EOF
protoPayload.@type="type.googleapis.com/google.cloud.audit.AuditLog"
protoPayload.metadata.@type="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
EOF
}

# Create an org level log sink
resource "google_logging_organization_sink" "bq_log_sink" {
  name   = "vpcsc-bq-sink"
  org_id = var.gcp_org_id

  # "bigquery.googleapis.com/projects/[PROJECT_ID]/datasets/[DATASET]"
  destination = "bigquery.googleapis.com/${google_bigquery_dataset.vpcsc_violations_dataset.id}"

  filter = trimspace(local.base_filter)

  bigquery_options {
    use_partitioned_tables = true
  }

  include_children = true
}

# This permission is required as documented here https://cloud.google.com/logging/docs/export/configure_export_v2#dest-auth
# Grant the org level service account the Big Query editor role
resource "google_bigquery_dataset_iam_member" "vpcsc_violations_dataset" {
  dataset_id = google_bigquery_dataset.vpcsc_violations_dataset.dataset_id
  role       = "roles/bigquery.dataEditor"
  member     = "serviceAccount:${local.org_service_account}"
}

# Create a BQ Data set where the log sink will export VPC SC logs
# This will then be visualised in Looker / Data Studio
resource "google_bigquery_dataset" "vpcsc_violations_dataset" {
  dataset_id    = "vpsc_violations_data"
  friendly_name = "vpsc-violations-data"
  description   = "VPC Service Control Violations Data"
  location      = "EU"

  # Make a group (or it could be a user but the former is preferred) the owner of the data set
  access {
    role           = "OWNER"
    group_by_email = "mygroup@acme.com"
  }

  # Grant the org service account writer permissions
  access {
    role          = "WRITER"
    user_by_email = local.org_service_account
  }

  access {
    role           = "READER"
    group_by_email = "anothergroup@acme.com"
  }
}

# Create a table from a view - this extracts nested fields from 
# the field protopayload_auditlog.metadataJson 
# this makes it easier to show information from this field in Data Studio/Looker charts
resource "google_bigquery_table" "cleaned_vpcsc_violations_table" {
  dataset_id = google_bigquery_dataset.vpcsc_violations_dataset.dataset_id
  table_id   = "cleaned_vpcsc_violations_table"

  view {
    query = templatefile("${path.module}/vpcsc-violations-bq-table.tpl", {
      project = local.bq_project
      dataset = google_bigquery_dataset.vpcsc_violations_dataset.dataset_id
      }
    )
    use_legacy_sql = false
  }
}
```

Here is some example SQL to create the table view:

```SQL
SELECT
   *,
    CASE
      WHEN REGEXP_CONTAINS(protopayload_auditlog.status.message,'(Dry Run Mode)*') THEN 'Dry Run'
      ELSE 'Enforced'
    END enforced_type,
    CASE
      WHEN REGEXP_CONTAINS(protopayload_auditlog.requestMetadata.callerIp, r"(^127\.)|(^10\.)|(^172\.1[6-9]\.)|(^172\.2[0-9]\.)|(^172\.3[0-1]\.)|(^192\.168\.)") OR protopayload_auditlog.requestMetadata.callerIp = 'private' THEN 1
      ELSE 0
    END internal_ip,
    REGEXP_EXTRACT(protopayload_auditlog.metadataJson, r'servicePerimeterName":"[a-zA-Z]+\/[\d]+\/[a-zA-Z]+\/([a-zA-Z_]+)') as perimeter,
    REGEXP_EXTRACT(protopayload_auditlog.metadataJson, r'violationReason":"([a-zA-Z_]+)') as violation_reason,    
FROM 
    `${project}.${dataset}.cloudaudit_googleapis_com_policy`
```

This is the same example query that Lorenzo provides in hist post - the only change is to permit underscores in the RegEx for the perimeter.

The Google [documentation](https://cloud.google.com/vpc-service-controls/docs/enable#analyze-violations) also have their own example:


```SQL
SELECT
    receiveTimestamp, #time of violation
    Resource.labels.service, #protected Google Cloud service being blocked
    protopayload_auditlog.methodName, #method name being called
    resource.labels.project_id as PROJECT, #protected project blocking the call
    protopayload_auditlog.authenticationInfo.principalEmail, #caller identity
    protopayload_auditlog.requestMetadata.callerIp, #caller IP
    CASE
      WHEN REGEXP_CONTAINS(protopayload_auditlog.requestMetadata.callerIp, r"(^127\.)|(^10\.)|(^172\.1[6-9]\.)|(^172\.2[0-9]\.)|(^172\.3[0-1]\.)|(^192\.168\.)") OR protopayload_auditlog.requestMetadata.callerIp = 'private' THEN 1
      ELSE 0
    END internal_ip, # Bool to indicate whether the caller IP is an RFC1918 address range or not
    JSON_EXTRACT(protopayload_auditlog.metadataJson, '$.dryRun') as DRYRUN, #dry-run indicator
    JSON_EXTRACT(protopayload_auditlog.metadataJson, '$.violationReason') as REASON, #reason for violation
    protopayload_auditlog.metadataJson, #raw violation entry
FROM
    `${project}.${dataset}.cloudaudit_googleapis_com_policy`
WHERE 
    JSON_EXTRACT(protopayload_auditlog.metadataJson, '$.dryRun') = "true" #ensure these are dry-run logs
```

The good thing about Google's version is that it uses the `as` keyword to create aliases for table columns that are more user friendly, so you do not need to rename them in Looker Studio. I have amended it to include the column that indicates if the IP is internal or not from Lorenzo's version.
