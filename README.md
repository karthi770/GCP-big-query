### Cloud Functions

![image](https://github.com/karthi770/Hosting-Wordpress-AWS/assets/102706119/a57316dd-018c-49e2-abfe-7bcc79758eef)
>[!important]
>Enable all these APIs before creating the functions

![image](https://github.com/karthi770/Hosting-Wordpress-AWS/assets/102706119/865172d7-fbff-471b-8ee9-559a0c8d7cf2)
![image](https://github.com/karthi770/Hosting-Wordpress-AWS/assets/102706119/c69e7610-6d6c-46b8-93ed-614e138f70e7)

![image](https://github.com/karthi770/Hosting-Wordpress-AWS/assets/102706119/6c3eb351-bc45-440f-8dfc-71925db7c680)
![image](https://github.com/karthi770/Hosting-Wordpress-AWS/assets/102706119/69d73472-170d-4436-abe1-6b1abdc89f64)
![image](https://github.com/karthi770/Hosting-Wordpress-AWS/assets/102706119/7f0a2f6f-fa7d-4aca-aef1-72801ebf6337)
![image](https://github.com/karthi770/Hosting-Wordpress-AWS/assets/102706119/11535341-b928-4cc1-85f5-b078c65c6020)

```python
#main.py

import pandas as pd
from pandas.io import gbq
from google.cloud import bigquery

'''
Python Dependencies to be installed

gcsfs
fsspec
pandas
pandas-gbq

'''

def hello_gcs(event, context):
    """Triggered by a change to a Cloud Storage bucket.
    Args:
         event (dict): Event payload.
         context (google.cloud.functions.Context): Metadata for the event.
    """

    lst = []
    file_name = event['name']
    table_name = file_name.split('.')[0]

    # Event,File metadata details writing into Big Query
    dct={
         'Event_ID':context.event_id,
         'Event_type':context.event_type,
         'Bucket_name':event['bucket'],
         'File_name':event['name'],
         'Created':event['timeCreated'],
         'Updated':event['updated']
        }
    lst.append(dct)
    df_metadata = pd.DataFrame.from_records(lst)
    df_metadata.to_gbq('gcp_dataeng.data_loading_metadata', 
                        project_id='cicd-gcp-412701', 
                        if_exists='append',
                        location='us')
    
    # Actual file data , writing to Big Query
    df_data = pd.read_csv('gs://' + event['bucket'] + '/' + file_name)

    df_data.to_gbq('gcp_dataeng.' + table_name, 
                        project_id='cicd-gcp-412701', 
                        if_exists='append',
                        location='us')
```
```python
#requirement.txt
gcsfs
fsspec
pandas
pandas-gbq
```

![image](https://github.com/karthi770/Hosting-Wordpress-AWS/assets/102706119/ebfe9ebc-c613-456a-a6f4-de219ded299e)
>[!important]
>Make sure the big query permissions are added to the project

![image](https://github.com/karthi770/Hosting-Wordpress-AWS/assets/102706119/524a5d50-08da-494b-ba74-af29e07b657e)
```python
#provider.tf
provider "google" {
  credentials = file(var.credentials)
  project     = var.project_id
  region      = var.region
}
```
```python
#main.tf
resource "google_project_service" "project" {
  project = var.project_id
  service = "storage.googleapis.com"
  disable_dependent_services = true
}

resource "google_storage_bucket" "data-pipeline" {
    name     = "data-eng-pipe"
    location = "us"
    uniform_bucket_level_access = true
}

resource "google_project_iam_binding" "project" {
  project = var.project_id
  role    = "roles/admin"
  members = ["serviceAccount:big-query@big-query-terraform.iam.gserviceaccount.com"]
}

resource "google_storage_bucket_iam_binding" "binding" {
    bucket = google_storage_bucket.data-pipeline.name
    role   = "roles/storage.admin"
    members = ["serviceAccount:big-query@big-query-terraform.iam.gserviceaccount.com"]
}
```
```python
#var.tf
variable "project_id" {
}
variable "region" {
}
variable "credentials" {
}
```
```python
#terraform.tfvars
project_id = "big-query-terraform"
region = "us-east1"
credentials = "../big-query-terraform-89a30e1306e1.json"
```

```python
resource "google_storage_bucket_object" "function_code" {
  name   = "main.zip"
  bucket = google_storage_bucket.cloud_function.name
  source = "../main.zip" # Add path to the zipped function source code
}

resource "google_storage_bucket_object" "requirements" {
  name   = "requirements.txt"
  bucket = google_storage_bucket.cloud_function.name
  source = "../requirements.txt" # Add path to the zipped function source code
}
```

![image](https://github.com/karthi770/Hosting-Wordpress-AWS/assets/102706119/15aecf3d-5f33-4cd9-811f-2ae2a01213eb)

```python
resource "google_project_iam_member" "gcs-pubsub-publishing" {
  project = var.project_id
  role    = "roles/pubsub.publisher"
  member  = var.serviceAccount
}
resource "google_service_account" "account" {
  account_id   = "gcf-sa"
  display_name = "Test Service Account - used for both the cloud function and eventarc trigger in the test"
}
# Permissions on the service account used by the function and Eventarc trigger

resource "google_project_iam_member" "invoking" {
  project    = var.project_id
  role       = "roles/run.invoker"
  member     = "serviceAccount:${google_service_account.account.email}"
  depends_on = [google_project_iam_member.gcs-pubsub-publishing]
}
resource "google_project_iam_member" "event-receiving" {
  project    = var.project_id
  role       = "roles/eventarc.eventReceiver"
  member     = "serviceAccount:${google_service_account.account.email}"
  depends_on = [google_project_iam_member.invoking]
}

resource "google_project_iam_member" "artifactregistry-reader" {
  project    = var.project_id
  role       = "roles/artifactregistry.reader"
  member     = "serviceAccount:${google_service_account.account.email}"
  depends_on = [google_project_iam_member.event-receiving]

resource "google_cloud_run_v2_service_iam_binding" "binding" {
  project = google_cloud_run_v2_service.default.project
  location = google_cloud_run_v2_service.default.location
  name = google_cloud_run_v2_service.default.name
  role = "roles/admin"
  members = [var.serviceAccount]
}

resource "google_cloud_run_v2_service" "default" {
  name     = "cloudrun-service"
  location = "us-central1"
  ingress = "INGRESS_TRAFFIC_ALL"
  template {
    containers {
      image = "us-docker.pkg.dev/cloudrun/container/hello"
      command = ["/server"]
    }
  }
}

resource "google_cloudfunctions2_function" "function" {
  depends_on = [
    google_project_iam_member.event-receiving,
    google_project_iam_member.artifactregistry-reader,
  ]
  name        = "gcf-function"
  location    = "us-central1"
  description = "a new function"


  build_config {
    runtime     = "python310"
    entry_point = "hello_gcs" # Set the entry point in the code
    environment_variables = {
      BUILD_CONFIG_TEST = "build_test"
    }
    source {
      storage_source {
        bucket = google_storage_bucket.cloud_function.name
        object = google_storage_bucket_object.function_code.name
      }
    }
  }

  service_config {
    max_instance_count = 3
    min_instance_count = 1
    available_memory   = "256M"
    timeout_seconds    = 60
    environment_variables = {
      SERVICE_CONFIG_TEST = "config_test"
      name = "PORT"
      value = "8080"
    }
    ingress_settings               = "ALLOW_ALL"
    all_traffic_on_latest_revision = true
    service_account_email          = google_service_account.account.email
  }
  event_trigger {
    trigger_region        = "us-central1" # The trigger must be in the same location as the bucket
    event_type            = "google.cloud.storage.object.v1.finalized"
    retry_policy          = "RETRY_POLICY_RETRY"
    service_account_email = google_service_account.account.email
    event_filters {
      attribute = "bucket"
      value     = google_storage_bucket.data-pipeline.name
    }
  }
}
```