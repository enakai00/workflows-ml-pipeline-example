# workflows-ml-pipeline-example

This example shows how you can use [Cloud Run](https://cloud.google.com/run) and [Cloud Workflows](https://cloud.google.com/workflows) to create a simple ML pipeline. The ML usecase is based on the [babyweight model example](https://github.com/GoogleCloudPlatform/training-data-analyst/blob/master/blogs/babyweight_keras/babyweight.ipynb).

## Deploy services

Set environment variables and create a storage bucket.

```shell
PROJECT_ID='[your project id]'
GIT_REPO="https://github.com/enakai00/workflows-ml-pipeline-example.git"
BUCKET=gs://$PROJECT_ID-pipeline

gcloud config set project $PROJECT_ID
gsutil mb $BUCKET
```

Enable APIs.

```shell
gcloud services enable run.googleapis.com
gcloud services enable workflows.googleapis.com
gcloud services enable cloudbuild.googleapis.com
gcloud services enable dataflow.googleapis.com
gcloud services enable ml.googleapis.com
```

Deploy services on Cloud Run.

```shell
cd $HOME/workflows-ml-pipeline-example/services/preprocess
gcloud builds submit --tag gcr.io/$PROJECT_ID/preprocess-service
gcloud run deploy preprocess-service \
  --image gcr.io/$PROJECT_ID/preprocess-service \
  --platform=managed --region=us-central1 \
  --no-allow-unauthenticated \
  --memory 512Mi \
  --set-env-vars "PROJECT_ID=$PROJECT_ID"

cd $HOME/workflows-ml-pipeline-example/services/train
gcloud builds submit --tag gcr.io/$PROJECT_ID/train-service
gcloud run deploy train-service \
  --image gcr.io/$PROJECT_ID/train-service \
  --platform=managed --region=us-central1 \
  --no-allow-unauthenticated \
  --memory 512Mi \
  --set-env-vars "PROJECT_ID=$PROJECT_ID,GIT_REPO=$GIT_REPO"
```

Get service URLs.

```shell
SERVICE_NAME="preprocess-service"
PREPROCESS_SERVICE_URL=$(gcloud run services list --platform managed \
    --format="table[no-heading](URL)" --filter="SERVICE:${SERVICE_NAME}")

SERVICE_NAME="train-service"
TRAIN_SERVICE_URL=$(gcloud run services list --platform managed \
    --format="table[no-heading](URL)" --filter="SERVICE:${SERVICE_NAME}")
```

## Test services using the curl command

Execute the data preprocessing dataflow job.

```shell
curl -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  -H "Content-Type: application/json" \
  -d "{\"limit\":\"1000\", \"outputDir\":\"$BUCKET/preproc\"}" \
  -s $PREPROCESS_SERVICE_URL/api/v1/preprocess | jq .
```

[Output]
```shell
{
  "jobId": "2020-12-13_23_57_52-4099585880410245609",
  "jobName": "preprocess-babyweight-054aeefe-16d2-4a26-a5c2-611a5ece1583",
  "outputDir": "gs://workflows-ml-pipeline-pipeline/preproc/054aeefe-16d2-4a26-a5c2-611a5ece1583"
}
```

The preprocessed data will be stored under `outputDir`.

```shell
OUTPUT_DIR="gs://workflows-ml-pipeline-pipeline/preproc/054aeefe-16d2-4a26-a5c2-611a5ece1583"
```

Check the job status.

```
JOB_ID="2020-12-13_23_57_52-4099585880410245609"
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  -s $PREPROCESS_SERVICE_URL/api/v1/job/$JOB_ID | jq .
```

[Output]
```shell
{
  "createTime": "2020-12-14T07:57:54.086857Z",
  "currentState": "JOB_STATE_RUNNING",
  "currentStateTime": "2020-12-14T07:57:59.416039Z",
...
}
```

Wait until the `currentState` becomes `JOB_STATE_DONE`. Then execute the model training job.

```shell
curl -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
-H "Content-Type: application/json" \
-d "{\"jobDir\": \"$BUCKET/trained_model\", \"dataDir\": \"$OUTPUT_DIR\", \"numTrainExamples\": 5000, \"numEvals\": 2, \"numEvalExamples\": 1000}" \
-s $TRAIN_SERVICE_URL/api/v1/train | jq .
```

[OUTPUT]
```
{
  "createTime": "2020-12-14T08:24:12Z",
  "etag": "zKM8N6bPpVk=",
  "jobId": "train_babyweight_e281aab4_5b4f_40cd_8fe3_f8290037b5fc",
  "state": "QUEUED",
  "trainingInput": {
    "args": [
      "--data-dir",
      "gs://workflows-ml-pipeline-pipeline/preproc/054aeefe-16d2-4a26-a5c2-611a5ece1583",
      "--num-train-examples",
      "5000",
      "--num-eval-examples",
      "1000",
      "--num-evals",
      "2",
      "--learning-rate",
      "0.0001"
    ],
    "jobDir": "gs://workflows-ml-pipeline-pipeline/trained_model/e281aab4-5b4f-40cd-8fe3-f8290037b5fc",
    "packageUris": [
      "gs://workflows-ml-pipeline-pipeline/trained_model/e281aab4-5b4f-40cd-8fe3-f8290037b5fc/trainer-0.0.0.tar.gz"
    ],
    "pythonModule": "trainer.task",
    "pythonVersion": "3.7",
    "region": "us-central1",
    "runtimeVersion": "2.2",
    "scaleTier": "BASIC_GPU"
  },
  "trainingOutput": {}
}
```

The trained model will be stored under `jobDir`.

```shell
JOB_DIR="gs://workflows-ml-pipeline-pipeline/trained_model/e281aab4-5b4f-40cd-8fe3-f8290037b5fc"
```

Check the job status.

```shell
JOB_ID="train_babyweight_e281aab4_5b4f_40cd_8fe3_f8290037b5fc"
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  -s $TRAIN_SERVICE_URL/api/v1/job/$JOB_ID | jq .
```

[OUTPUT]
```
{
  "createTime": "2020-12-14T08:24:12Z",
  "etag": "rW+uQQbA6bM=",
  "jobId": "train_babyweight_e281aab4_5b4f_40cd_8fe3_f8290037b5fc",
  "state": "PREPARING",
...
}
```

Wait until `state` becomes `SUCCEEDED`. Then execute the model deployment job.

```shell
curl -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  -H "Content-Type: application/json" \
  -d "{\"modelName\": \"babyweight_model\", \"versionName\": \"v1\", \"deploymentUri\": \"$JOB_DIR/export\"}" \
 -s $TRAIN_SERVICE_URL/api/v1/deploy | jq .
```

[OUTPUT]
```shell
{
  "metadata": {
    "@type": "type.googleapis.com/google.cloud.ml.v1.OperationMetadata",
    "createTime": "2020-12-14T08:34:36Z",
    "modelName": "projects/workflows-ml-pipeline/models/babyweight_model",
    "operationType": "CREATE_VERSION",
    "version": {
      "createTime": "2020-12-14T08:34:35Z",
      "deploymentUri": "gs://workflows-ml-pipeline-pipeline/trained_model/e281aab4-5b4f-40cd-8fe3-f8290037b5fc/export",
      "etag": "BlXqEgx9VQg=",
      "framework": "TENSORFLOW",
      "machineType": "mls1-c1-m2",
      "name": "projects/workflows-ml-pipeline/models/babyweight_model/versions/v1",
      "pythonVersion": "3.7",
      "runtimeVersion": "2.2"
    }
  },
  "name": "projects/workflows-ml-pipeline/operations/create_babyweight_model_v1-1607934875576"
}
```

## Automate the pipeline with Cloud Workflows

Create a service account to invoke services on Cloud Run.

```shell
SERVICE_ACCOUNT_NAME="cloud-run-invoker"
SERVICE_ACCOUNT_EMAIL=${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com
gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME \
  --display-name "Cloud Run Invoker"
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$SERVICE_ACCOUNT_EMAIL \
  --role=roles/run.invoker
```

Deploy the workflow.

```shell
cd $HOME/workflows-ml-pipeline-example/workflows
cp ml_workflow.yaml.template ml_workflow.yaml
sed -i "s#PREPROCESS-SERVICE-URL#${PREPROCESS_SERVICE_URL}#" ml_workflow.yaml
sed -i "s#TRAIN-SERVICE-URL#${TRAIN_SERVICE_URL}#" ml_workflow.yaml
gcloud beta workflows deploy ml_workflow \
  --source=ml_workflow.yaml \
  --service-account=$SERVICE_ACCOUNT_EMAIL
```

Execute a workflow job.

```shell
gcloud beta workflows execute ml_workflow \
  --data="{\"limit\": 1000, \"bucket\": \"$BUCKET\", \"numTrainExamples\": 5000, \"numEvals\": 2, \"numEvalExamples\": 1000, \"modelName\": \"babyweight_model\", \"versionName\": \"v2\"}"
```
