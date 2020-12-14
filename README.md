# workflows-pipeline-example

```shell
PROJECT_ID='[your project id]'
GIT_REPO="https://github.com/enakai00/workflows-ml-pipeline-example.git"
BUCKET=gs://$PROJECT_ID-pipeline
gsutil mb $BUCKET
```

```shell
cd $HOME/workflows-pipeline-example/services/preprocess
gcloud builds submit --tag gcr.io/$PROJECT_ID/preprocess-service
gcloud run deploy preprocess-service \
  --image gcr.io/$PROJECT_ID/preprocess-service \
  --platform=managed --region=us-central1 \
  --no-allow-unauthenticated \
  --memory 512Mi \
  --set-env-vars "PROJECT_ID=$PROJECT_ID"

cd $HOME/workflows-pipeline-example/services/train
gcloud builds submit --tag gcr.io/$PROJECT_ID/train-service
gcloud run deploy train-service \
  --image gcr.io/$PROJECT_ID/train-service \
  --platform=managed --region=us-central1 \
  --no-allow-unauthenticated \
  --memory 512Mi \
  --set-env-vars "PROJECT_ID=$PROJECT_ID,GIT_REPO=$GIT_REPO"
```

```shell
SERVICE_NAME="preprocess-service"
PREPROCESS_SERVICE_URL=$(gcloud run services list --platform managed \
    --format="table[no-heading](URL)" --filter="SERVICE:${SERVICE_NAME}")

SERVICE_NAME="train-service"
TRAIN_SERVICE_URL=$(gcloud run services list --platform managed \
    --format="table[no-heading](URL)" --filter="SERVICE:${SERVICE_NAME}")
```

```shell
curl -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  -H "Content-Type: application/json" \
  -d "{\"limit\":\"1000\", \"outputDir\":\"$BUCKET/preproc\"}" \
  -s $PREPROCESS_SERVICE_URL/api/v1/preprocess | jq .
```

[Output]
```shell
{
  "jobId": "2020-12-13_00_42_47-18156259136194922599",
  "jobName": "preprocess-babyweight-b7492423-c4fe-4866-93c2-54c782168cd9",
  "outputDir": "gs://babyweight-keras2-pipeline/preproc/b7492423-c4fe-4866-93c2-54c782168cd9"
}
```

```shell
OUTPUT_DIR="gs://babyweight-keras2-pipeline/preproc/b7492423-c4fe-4866-93c2-54c782168cd9"
JOBID="2020-12-13_00_42_47-18156259136194922599"
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  -s $PREPROCESS_SERVICE_URL/api/v1/job/$JOBID | jq .
```

[Output]
```shell
{
  "createTime": "2020-12-13T22:22:41.752648Z",
  "currentState": "JOB_STATE_DONE",
  "currentStateTime": "2020-12-13T22:28:17.997861Z",
...
}
```

```shell
curl -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
-H "Content-Type: application/json" \
-d "{\"jobDir\": \"$BUCKET/trained_model\", \"dataDir\": \"$OUTPUT_DIR\", \"numTrainExamples\": 1000, \"numEvals\": 1, \"numEvalExamples\": 1000}" \
-s $TRAIN_SERVICE_URL/api/v1/train | jq .
```

[OUTPUT]
```
{
  "createTime": "2020-12-14T01:02:10Z",
  "etag": "TdMwRe0dgG8=",
  "jobId": "train_babyweight_1bcc7451_e887_4fb9_acf2_7ce3cda4b363",
  "state": "QUEUED",
  "trainingInput": {
    "args": [
      "--data-dir",
      "gs://babyweight-keras2-pipeline/preproc/65849499-0ba5-44e0-b32e-d4520a8a18bc",
      "--num-train-examples",
      "1000",
      "--num-eval-examples",
      "1000",
      "--num-evals",
      "1",
      "--learning-rate",
      "0.0001"
    ],
    "jobDir": "gs://babyweight-keras2-pipeline/trained_model/1bcc7451-e887-4fb9-acf2-7ce3cda4b363",
    "packageUris": [
      "gs://babyweight-keras2-pipeline/trained_model/1bcc7451-e887-4fb9-acf2-7ce3cda4b363/trainer-0.0.0.tar.gz"
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

```shell
JOBID="train_babyweight_1bcc7451_e887_4fb9_acf2_7ce3cda4b363"
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  -s $TRAIN_SERVICE_URL/api/v1/job/$JOBID | jq .
```

[OUTPUT]
```
{
  "createTime": "2020-12-14T01:02:10Z",
  "etag": "hzA1lBs5RHE=",
  "jobId": "train_babyweight_1bcc7451_e887_4fb9_acf2_7ce3cda4b363",
  "startTime": "2020-12-14T01:07:08Z",
  "state": "SUCCEEDED",
...
    "jobDir": "gs://babyweight-keras2-pipeline/trained_model/1bcc7451-e887-4fb9-acf2-7ce3cda4b363",
...
}
```

```shell
JOBDIR="gs://babyweight-keras2-pipeline/trained_model/1bcc7451-e887-4fb9-acf2-7ce3cda4b363"
curl -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  -H "Content-Type: application/json" \
  -d "{\"modelName\": \"babyweight_model\", \"versionName\": \"v1\", \"deploymentUri\": \"$JOBDIR/export\"}" \
 -s $TRAIN_SERVICE_URL/api/v1/deploy | jq .
```

[OUTPUT]
```shell
{
  "metadata": {
    "@type": "type.googleapis.com/google.cloud.ml.v1.OperationMetadata",
    "createTime": "2020-12-14T01:11:02Z",
    "modelName": "projects/babyweight-keras2/models/babyweight_model",
    "operationType": "CREATE_VERSION",
    "version": {
      "createTime": "2020-12-14T01:11:01Z",
      "deploymentUri": "gs://babyweight-keras2-pipeline/trained_model/1bcc7451-e887-4fb9-acf2-7ce3cda4b363/export",
      "etag": "24SGjgPmN/k=",
      "framework": "TENSORFLOW",
      "machineType": "mls1-c1-m2",
      "name": "projects/babyweight-keras2/models/babyweight_model/versions/v1",
      "pythonVersion": "3.7",
      "runtimeVersion": "2.2"
    }
  },
  "name": "projects/babyweight-keras2/operations/create_babyweight_model_v1-1607908261193"
}
```



```shell
SERVICE_ACCOUNT_NAME="cloud-run-invoker"
SERVICE_ACCOUNT_EMAIL=${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com
gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME \
  --display-name "Cloud Run Invoker"
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$SERVICE_ACCOUNT_EMAIL \
  --role=roles/run.invoker
```

```shell
cd $HOME/workflows-pipeline-example/workflows
cp ml_workflow.yaml.template ml_workflow.yaml
sed -i "s#PREPROCESS-SERVICE-URL#${PREPROCESS_SERVICE_URL}#" ml_workflow.yaml
sed -i "s#TRAIN-SERVICE-URL#${TRAIN_SERVICE_URL}#" ml_workflow.yaml
gcloud beta workflows deploy ml_workflow \
  --source=ml_workflow.yaml \
  --service-account=$SERVICE_ACCOUNT_EMAIL
```

```shell
gcloud beta workflows execute ml_workflow \
  --data="{\"limit\": 1000, \"bucket\": \"$BUCKET\", \"numTrainExamples\": 1000, \"numEvals\": 1, \"numEvalExamples\": 1000, \"modelName\": \"babyweight\", \"versionName\": \"v1\"}"
```
