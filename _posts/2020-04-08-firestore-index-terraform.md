---
title: "Importing firestore index to terraform"
layout: post
date: 2020-04-08 12:43
headerImage: false
tag:
- terraform
- firestore
- google cloud
category: blog
author: kosyanyanwu
description: Import existing firestore index to be managed by terraform
---
## Introduction
[Google Cloud Firestore](https://firebase.google.com/docs/firestore/query-data/indexing) ensures query performance by requiring an index for every query. Index creation takes a few minutes, depending on how much data needs to be updated. The more documents that match the fields being indexed, the longer creation takes. Also, for each unique collection ID, you can have only one index build in progress. Multiple index builds on the same collection ID complete sequentially.

Terraform has a [google_firestore_index](https://www.terraform.io/docs/providers/google/r/firestore_index.html) resource used to manage composite indexes, with a default timeout of 10 mins. However, it supports the use of a timeout configuration option, where you can specify your own default. In a situation where you do not know how long the index build will take, and you use a lesser default, terraform will time out before the index build completes. If you try to apply terraform again, it will give the following error:

```
Error: Error creating Index: googleapi: Error 409: index already exists
```
This error is due to the fact that the index already exists as it is already being created, but you don't have the final state of the resource when creation is complete. In this case, [terraform import](https://www.terraform.io/docs/import/index.html) comes to the rescue. Terraform is able to import existing infrastructure. This allows you to take resources you have created by some other means and bring it under Terraform management.

This article explains how you can import the existing firestore index infrastructure to be managed by Terraform. As at the time of writing this, the current implementation of Terraform import can only import resources into the state. It does not generate configuration.

## Import existing firestore index infrastructure

Assume you have a configuration like this:
```tf
resource "google_firestore_index" "my-index" {
  project = "my-project-name"

  collection = "chatrooms"

  fields {
    field_path = "name"
    order      = "ASCENDING"
  }

  fields {
    field_path = "description"
    order      = "DESCENDING"
  }

  fields {
    field_path = "__name__"
    order      = "DESCENDING"
  }
}
```
And you want to import the state for it in this form
```sh
terraform import 'google_firestore_index.my-index' <name>
```
where [name](https://www.terraform.io/docs/providers/google/r/firestore_index.html#name) is a server defined name for this index with the format `projects/{project}/databases/{database}/collectionGroups/{collection}/indexes/{server_generated_id}`.

In this format, `{project}` is your GCP project ID, `{database}` is the name of the database e.g default, `{collection}` is the name of the collection in the example configuration above i.e `chatrooms`, and `{server_generated_id}` is the `NAME` gotten when you do:
```sh
$ gcloud firestore indexes composite list
┌──────────────┬──────────────────┬─────────────┬───────┬────────────────────────────────┬────────────┬──────────────┐
│     NAME     │ COLLECTION_GROUP │ QUERY_SCOPE │ STATE │          FIELD_PATHS           │   ORDER    │ ARRAY_CONFIG │
├──────────────┼──────────────────┼─────────────┼───────┼────────────────────────────────┼────────────┼──────────────┤
│ CICAgPilZUK  │ chatrooms        │ COLLECTION  │ READY │ name                           │ ASCENDING  │              │
│              │                  │             │       │ description                    │ ASCENDING  │              │
├──────────────┼──────────────────┼─────────────┼───────┼────────────────────────────────┼────────────┼──────────────┤

```
So we end up having:
```sh
$ terraform import 'google_firestore_index.my-index' 'projects/your_gcp_project_id/databases/(default)/collectionGroups/chatrooms/indexes/CICAgPilZUK'

google_firestore_index.my-index: Importing from ID "projects/your_gcp_project_id/databases/(default)/collectionGroups/chatrooms/indexes/CICAgPilZUK"...
google_firestore_index.my-index: Import prepared!
Prepared google_firestore_index for import
google_firestore_index.my-index: Refreshing state... [id=projects/your_project_id/databases/(default)/collectionGroups/chatrooms/indexes/CICAgPilZUK]

Import successful!

The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.
```
This should import your existing resource into terraform state, and when you do terraform plan, you should not see any changes to apply.
