---
title: "Automating logs export to BigQuery with Terraform"
layout: post
date: 2020-12-11 22:05
headerImage: false
tag:
- terraform
- google cloud
- bigquery
- logsink
category: blog
author: kosyanyanwu
description: How to automate logs export to BigQuery on Google Cloud with Terraform
---
## Introduction
On [Google cloud](https://cloud.google.com/logging/docs/export/configure_export_v2), you export logs by creating one or more sinks that include a logs filter and an export destination. As Cloud Logging receives new log entries, they are compared against each sink. If a log entry matches a sink's filter, then a copy of the log entry is written to the export destination.

This article describes how you can automate logs exports to [BigQuery](https://cloud.google.com/bigquery/docs) with Terraform.


## Create BigQuery dataset
Using the [google_bigquery_dataset](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/bigquery_dataset) resource, create a BigQuery dataset.
```tf
    resource "google_bigquery_dataset" "dataset" {
        dataset_id                  = "example_dataset"
        friendly_name               = "test"
        description                 = "This is a test description"
        location                    = "EU"

        labels = {
            env = "default"
        }

        access {
            role          = "OWNER"
            user_by_email = google_service_account.bqowner.email
        }

        access {
            role   = "READER"
            domain = "hashicorp.com"
        }
    }
```

## Create log sink
You can create a [project-level](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/logging_project_sink), [organization-level](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/logging_organization_sink) or [folder-level](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/logging_folder_sink) logging sink. In this article, we will be creating a project-level logging sink using the [google_logging_project_sink](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/logging_project_sink) resource.

```tf
resource "google_logging_project_sink" "log_sink" {
    name        = "my_log_sink"
    description = "some explaination on what this is"
    destination = "bigquery.googleapis.com/projects/[PROJECT_ID]/datasets/${google_bigquery_dataset.dataset.dataset_id}"
    filter      = "resource.type = gce_instance AND resource.labels.instance_id = mylabel"

    # Whether or not to create a unique identity associated with this sink.
    unique_writer_identity = true

    bigquery_options {
    # options associated with big query
    # Refer to the resource docs for more information on the options you can use
  }
}
```

## Grant unique writer access to BigQuery
Because the sink above uses a `unique_writer`, we must grant that writer [bigquery.dataEditor](https://cloud.google.com/iam/docs/understanding-roles#bigquery-roles) role for it to access BigQuery. You can use the [google_project_iam_member](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project_iam#google_project_iam_member) resource.

```tf
resource "google_project_iam_member" "log_writer" {
    role = "roles/bigquery.dataEditor"
    member = google_logging_project_sink.log_sink.writer_identity
}
```

## Conclusion
After applying the steps above in a terraform file, in a short while, you should see a table created automatically in the BigQuery dataset with the logs exported to it.
