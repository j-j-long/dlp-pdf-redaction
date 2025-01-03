# Solution Guide
This solution provides an automated, serverless way to redact sensitive data from PDF files using Google Cloud Services like [Data Loss Prevention (DLP)](https://cloud.google.com/dlp), [Cloud Workflows](https://cloud.google.com/workflows), and [Cloud Run](https://cloud.google.com/run).

## DSAR Config
Custom config of the dlp api to meet the needs of the DSAR are supplied to redact some but not all pii.

## Solution Architecture Diagram
The image below describes the solution architecture of the pdf redaction process.

![Architecture Diagram](./architecture-diagram.png)

## Workflow Steps
The workflow consists of the following steps:
1. The user uploads a PDF file to a GCS bucket
1. A Workflow is triggered by [EventArc](https://cloud.google.com/eventarc/docs). This workflow orchestrates the PDF file redaction consisting of the following steps:
    - Split the PDF into single pages, convert pages into images, and store them in a working bucket
    - Redact each image using DLP Image Redact API
    - Assemble back the PDF file from the list of redacted images and store it on GCS (output bucket)
    - Write redacted quotes (findings) to BigQuery

# Deploy PDF Redaction app
The `terraform` folder contains the code needed to deploy the PDF Redaction application.

## What resources are created?
Main resources:
- Workflow
- CloudRun services for each component with its service accounts and permissions
  1. `pdf-spliter` - Split PDF into single-page image files
  1. `dlp-runner` - Runs each page file through DLP to redact sensitive information
  1. `pdf-merger` - Assembles back the pages into a single PDF
  1. `findings-writer` - Writes findings into BigQuery
- Cloud Storage buckets
  - *Input Bucket* - bucket where the original file is stored
  - *Working Bucket* - a working bucket in which all temp files will be stored as throughout the different workflow stages
  - *Output Bucket* - bucket where the redacted file is stored
- DLP template where InfoTypes and rules are specified. You can modify the `dlp.tf` file to specify your own INFO_TYPES and Rule Sets (refer to [terraform documentation for dlp templates](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/data_loss_prevention_inspect_template))
- BigQuery dataset and table where findings will be written

## How to deploy?
The following steps should be executed in Cloud Shell in the Google Cloud Console. 

### 1. Create a project and enable billing
Follow the steps in [this guide](https://cloud.google.com/resource-manager/docs/creating-managing-projects).

### 2. Get the code
Clone this github repository go to the root of the repository.

``` 
git clone https://github.com/GoogleCloudPlatform/dlp-pdf-redaction
cd dlp-pdf-redaction
```

### 3. Build images for Cloud Run
You will first need to build the docker images for each microservice.

```
PROJECT_ID=[YOUR_PROJECT_ID]
PROJECT_NUMBER=$(gcloud projects list --filter="PROJECT_ID=$PROJECT_ID" --format="value(PROJECT_NUMBER)")
REGION=europe-west2
DOCKER_REPO_NAME=pdf-redaction-docker-repo
CLOUD_BUILD_SERVICE_ACCOUNT=cloudbuild-sa

# Enable required APIs
gcloud services enable cloudbuild.googleapis.com artifactregistry.googleapis.com --project $PROJECT_ID

# Create a Docker image repo to store apps docker images
gcloud artifacts repositories create $DOCKER_REPO_NAME --repository-format=docker --description="PDF Redaction Docker Image repository" --project $PROJECT_ID --location=$REGION

# Create Service Account for CloudBuild and grant required roles
gcloud iam service-accounts create $CLOUD_BUILD_SERVICE_ACCOUNT \
  --description="Service Account for CloudBuild created by PDF Redaction solution" \
  --display-name="CloudBuild SA (PDF Readaction)"
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$CLOUD_BUILD_SERVICE_ACCOUNT@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/cloudbuild.serviceAgent"
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$CLOUD_BUILD_SERVICE_ACCOUNT@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectUser"

# Build docker images of the app and store them in artifact registry repo
gcloud builds submit \
  --config ./build-app-images.yaml \
  --substitutions _REGION=$REGION,_DOCKER_REPO_NAME=$DOCKER_REPO_NAME \
  --service-account=projects/$PROJECT_ID/serviceAccounts/$CLOUD_BUILD_SERVICE_ACCOUNT@$PROJECT_ID.iam.gserviceaccount.com \
  --default-buckets-behavior=regional-user-owned-bucket \
  --project $PROJECT_ID
```
Note: If you receive a pop-up for permissions, you can authorize gcloud to request your credentials an make a GCP API call.


The above command will build 4 docker images and push them into Google Container Registry (GCR). Run the following command and confirm that the images are present in GCR.

```
gcloud artifacts docker images list $REGION-docker.pkg.dev/$PROJECT_ID/$DOCKER_REPO_NAME
```

### 4. Deploy the infrastructure using Terraform

This terraform deployment requires the following variables. 

- project_id            = "YOUR_PROJECT_ID"
- region                = "YOUR_REGION_REGION"
- docker_repo_name      = "DOCKER_REPO_NAME"
- wf_region             = "YOUR_WORKFLOW_REGION"

From the root folder of this repo, run the following commands:
```
export TF_VAR_project_id=$PROJECT_ID
export TF_VAR_region=$REGION
export TF_VAR_wf_region=$REGION
export TF_VAR_docker_repo_name=$DOCKER_REPO_NAME

terraform -chdir=terraform init
terraform -chdir=terraform apply -auto-approve
```

**Notes:**
  * If you get an error related to `eventarc` or `worklflows` provisioning, just give it a few seconds and rerun the `terraform -chdir=terraform apply -auto-approve` command. Explanation: Terraform enables some services like `eventarc` an `workflows` that might take a couple of minutes to finish provisioning resources and configuring permissions, simply re-runing the apply command should fix the issue.
  * Region and Workflow region both default to `us-central1`. If you wish to deploy the resources in a different region, specify the `region` and the `wf_region` variables (ie. using `TF_VAR_region` and `TF_VAR_wf_region`). Cloud Workflows is only available in specific regions, for more information check the [documentation](https://cloud.google.com/workflows/docs/locations).
  * If you come across an issue please check the [Issues section](https://github.com/GoogleCloudPlatform/dlp-pdf-redaction/issues). If your issue is not listed there, please report it as a new issue.



### 5. Take note of Terraform Outputs

Once terraform finishes provisioning all resources, you will see its outputs. Please take note of `input_bucket` and `output_bucket` buckets. Files uploaded to the `input_bucket` bucket will be automatically processed and the redacted files will be written to the `output_bucket` bucket.
If you missed the outputs from the first run, you can list the outputs by running

```
terraform -chdir=terraform output
```

### 6. Test

Use the command below to upload the test file into the `input_bucket`. After a few seconds, you should see a redacted PDF file in the `output_bucket`.
```
gsutil cp ./test_file.pdf [INPUT_BUCKET_FROM_OUTPUT e.g. gs://pdf-input-bucket-xxxx]
```

If you are curious about the behind the scenes, try:
- Checkout the Redacted file in the `output_bucket`.

  ```
  gsutil ls [OUTPUT_BUCKET_FROM_OUTPUT e.g. gs://pdf-output-bucket-xxxx]
  ```

- Download the redacted pdf file, open it with your preferred pdf reader, and search for text in the PDF file.
- Looking into [Cloud Workflows](https://console.cloud.google.com/workflows) in the GCP web console. You will see that a workflow execution was triggered when you uploaded the file to GCS.
- Explore the `pdf_redaction_xxxx` dataset in BigQuery and check out the metadata that was inserted into the `findings` table.
