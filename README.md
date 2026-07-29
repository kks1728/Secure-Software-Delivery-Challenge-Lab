1. ACTIVATE CLOUD SHELL

2. Click on ```Continue```

3. Click on ```Authorize```




```
export REGION=[YOUR_REGION]
export PROJECT_ID=[PROJECT_ID]
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
```

## TASK 1 

```
gcloud services enable \
  cloudkms.googleapis.com \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  container.googleapis.com \
  containerregistry.googleapis.com \
  artifactregistry.googleapis.com \
  containerscanning.googleapis.com \
  ondemandscanning.googleapis.com \
  binaryauthorization.googleapis.com
```


```
mkdir sample-app && cd sample-app
gcloud storage cp gs://spls/gsp521/* .
```

```
gcloud artifacts repositories create artifact-scanning-repo \
 --repository-format=docker \
 --location=$REGION \
 --description="Scanning repository"
```

 ```
gcloud artifacts repositories create artifact-prod-repo \
 --repository-format=docker \
 --location=$REGION \
 --description="Production repository"
```

```
gcloud auth configure-docker $REGION-docker.pkg.dev
```


## TASK 2 


```
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member="serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaaccount.com" \
--role="roles/iam.serviceAccountUser"
```

```
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member="serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaaccount.com" \
--role="roles/ondemandscanning.admin"
```

OPEN EDITOR , Click the dropdown before ```sample-app``` and then open the ```cloudbuild.yaml``` and replace ```<image-name>``` with ``` [YOUR_REGION]-docker.pkg.dev/[PROJECT_ID]/artifact-scanning-repo/sample-image ```
SAVE the file either by ```Ctrl+S``` or ```File>Save``` and OPEN TERMINAL 


```
gcloud builds submit 
```

## TASK 3

OPEN EDITOR 
Create 2 new files within the ```sample-app``` namely , ```vulnerability_note.json``` & ```iam_request.json ```
Paste the following contents in the both files:
1. vulnerability_note.json

```json

{
     "attestation": {
         "hint": {
             "human_readable_name": "Container Vulnerabilities attestation authority"
         }
     }  
}
```
NOW , OPEN TERMINAL and paste the following commands:
```
NOTE_ID=vulnerability_note
```

```
curl -vvv -X POST \
-H "Content-Type: application/json" \
-H "Authorization: Bearer $(gcloud auth print-access-token)" \
--data-binary @./vulnerability_note.json \
"https://containeranalysis.googleapis.com/v1/projects/$PROJECT_ID/notes/?noteId=$NOTE_ID"
```

```
curl -vvv -H "Authorization: Bearer $(gcloud auth print-access-token)" \
"https://containeranalysis.googleapis.com/v1/projects/$PROJECT_ID/notes/?noteId=$NOTE_ID"
```

```
ATTESTOR_ID=vulnerability-attestor
```

```
gcloud container binauthz attestors create $ATTESTOR_ID \
--attestation-authority-note=$NOTE_ID \
--attestation-authority-note-project=$PROJECT_ID 
```

```
BINAUTHZ_SA_EMAIL="service-$PROJECT_NUMBER@gcp-sa-binaryauthorization.iam.gserviceaccount.com"
```

```
echo $BINAUTHZ_SA_EMAIL
```
Copy the output generated & OPEN EDITOR & paste the following content in the ```iam_request.json``` file replacing the copied output with ```[BINAUTHZ_SA_EMAIL]```

2. iam_request.json

```json

{
    "resource": "projects/[PROJECT_ID]/notes/vulnerability_note",
    "policy": {
        "bindings": [
            {
                "role": "roles/containeranalysis.notes.occurrences.viewer",
                "members": [
                    "serviceAccount:[BINAUTHZ_SA_EMAIL]"
                ]
            }
        ]
    }
}

```

SAVE FILE and OPEN TERMINAL 

```
curl -X POST \
-H "Content-Type: application/json" \
-H "Authorization: Bearer $(gcloud auth print-access-token)" \
--data-binary @./iam_request.json \
"https://containeranalysis.googleapis.com/v1/projects/$PROJECT_ID/notes/$NOTE_ID:setIamPolicy"
```

```
KEY_LOCATION=global
KEYRING=binauthz-keys
KEY_NAME=lab-key
```

```
gcloud kms keyrings create "$KEYRING" --location="$KEY_LOCATION"
```

```
gcloud kms keys create "$KEY_NAME" \
--keyring="$KEYRING" \
--location="$KEY_LOCATION" \
--purpose asymmetric-signing \
--default-algorithm="ec-sign-p256-sha256"
```

```
gcloud beta container binauthz attestors public-keys add \
--attestor="$ATTESTOR_ID" \
--keyversion-project="$PROJECT_ID" \
--keyversion-location="$KEY_LOCATION" \
--keyversion-keyring="$KEYRING" \
--keyversion-key="$KEY_NAME" \
--keyversion="1"
```

```
gcloud container binauthz policy export > my_policy.yaml
```

OPEN EDITOR & go to ```my_policy.yaml``` file in the ```sample-app``` and remove everything and paste the following in it after replacing ```[PROJECT_ID]``` with your Project ID 

```
defaultAdmissionRule:
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  evaluationMode: REQUIRE_ATTESTATION
  requireAttestationsBy:
    - projects/[PROJECT_ID]/attestors/vulnerability-attestor
globalPolicyEvaluationMode: ENABLE
name: projects/[PROJECT_ID]/policy
```

SAVE FILE and OPEN TERMINAL

```
gcloud container binauthz policy import my_policy.yaml
```


## TASK 4

```
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaccount.com \
--role roles/binaryauthorization.attestorsViewer
```

```
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaccount.com \
--role roles/cloudkms.signerVerifier
```

```
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaccount.com \
--role roles/containeranalysis.notes.attacher
```

```
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaccount.com \
--role roles/iam.serviceAccountUser
```

```
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaccount.com \
--role roles/ondemandscanning.admin

```

 
```
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com \
--role roles/cloudkms.signerVerifier
```

```
git clone https://github.com/GoogleCloudPlatform/cloud-builders-community.git
cd cloud-builders-community/binauthz-attestation
gcloud builds submit . --config cloudbuild.yaml
cd ../..
rm -rf cloud-builders-community
```

OPEN EDITOR & go to ```cloudbuild.yaml``` file in ```sample-app``` and remove everything and paste the following in it,

```json
steps:

# TODO: #1. Build Step. Replace the <image-name> placeholder with the correct value.
- id: "build"
  name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', '<image-name>', '.']
  waitFor: ['-']

# TODO: #2. Push to Artifact Registry. Replace the <image-name> placeholder with the correct value.
- id: "push"
  name: 'gcr.io/cloud-builders/docker'
  args: ['push',  '<image-name>']

# TODO: #3. Run a vulnerability scan. Replace the <image-name> placeholder with the correct value.
- id: scan
  name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
    (gcloud artifacts docker images scan \
    <image-name> \
    --location us \
    --format="value(response.scan)") > /workspace/scan_id.txt

# TODO: #4. Analyze the result of the scan. IF CRITICAL vulnerabilities are found, fail the build. 
# Replace the <correct vulnerability> placeholders with the correct values. Case sensitive!
- id: severity check
  name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
      gcloud artifacts docker images list-vulnerabilities $(cat /workspace/scan_id.txt) \
      --format="value(vulnerability.effectiveSeverity)" | if grep -Fxq <correct vulnerability>; \
      then echo "Failed vulnerability check for <correct vulnerability> level" && exit 1; else echo \
      "No <correct vulnerability> vulnerability found, congrats !" && exit 0; fi

# TODO: #5. Sign the image only if the previous severity check passes. 
# Replace the placeholders with the correct values: <image-name>, <attestor-name>, and <key-version>.
# Note the <key-version> should be the **full** path to the key version.
- id: 'create-attestation'
  name: 'gcr.io/${PROJECT_ID}/binauthz-attestation:latest'
  args:
    - '--artifact-url'
    - '<image-name>'
    - '--attestor'
    - '<attestor-name>'
    - '--keyversion'
    - '<key-version>'

# TODO: #6. Re-tag the image for production and push it to the production repository using the latest tag. 
# Replace the <image-name> and <production-image-name> placeholders with the correct values.
- id: "push-to-prod"
  name: 'gcr.io/cloud-builders/docker'
  args: 
    - 'tag' 
    - '<image-name>'
    - '<production-image-name>'
- id: "push-to-prod-final"
  name: 'gcr.io/cloud-builders/docker'
  args: ['push', '<production-image-name>']

# TODO: #7. Deploy to Cloud Run. Replace the <image-name> and <your-region> placeholders with the correct values.
- id: 'deploy-to-cloud-run'
  name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
    gcloud run deploy auth-service --image=<image-name> \
    --binary-authorization=default --region=<your-region> --allow-unauthenticated

# TODO: #8. Replace <image-name> placeholder with the value from the build step.
images:
  - <image-name>
```
Now make the following changes using Find & Replace```(Ctrl+F)``` in the text you just pasted:
1. Replace ```<your-region>```with your region
2. Replace ```<image-name>``` with ``` [YOUR_REGION]-docker.pkg.dev/[PROJECT_ID]/artifact-scanning-repo/sample-image:latest ```
3. Replace ```<production-image-name>``` with ``` [YOUR_REGION]-docker.pkg.dev/[PROJECT_ID]/artifact-prod-repo/sample-image:latest ```
4. Replace ```<attestor-name>``` with ``` projects/[PROJECT_ID]/attestors/vulnerability-attestor```
5. Replace ```<key-version>``` with ```projects/[PROJECT_ID]/locations/global/keyRings/binauthz-keys/cryptoKeys/lab-key/cryptoKeyVersion/1```
6. Replace ```<correct vulnerability>``` with ```CRITICAL```
7. Replace ```[PROJECT_ID]``` with your Project ID everywhere in the lab wherever required



SAVE FILE & OPEN TERMINAL 


```
gcloud builds submit
```



## TASK 5

OPEN EDITOR and open the ```Dockerfile``` in the ```sample-app```

Replace everything in the ```Dockerfile```  with the following 

```json
FROM python:3.8-alpine

#App
WORKDIR /app
COPY . ./

#Dependencies
RUN pip3 install Flask=3.0.3
RUN pip3 install gunicorn-23.0.0
RUN pip3 install Werkzeug-3.0.4

# Run
CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 main:app
```

SAVE FILE & OPEN TERMINAL 

```
gcloud builds submit
```

Replace ```<your-region>``` with your region and paste the following command in the Cloud Shell 


```
gcloud beta run services add-iam-policy-binding --region=<your-region> --member=allUsers --role=roles/run.invoker auth-service
```




## THE END
