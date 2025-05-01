This blog takes you through the detailed creation of a DevOps pipeline that automates the process of building an image, pushing it to a registry, and deploying it to Cloud Run (for real-time applications). This automation is triggered by pushing local code to a remote GitHub repository, utilizing Workload Identity Federation (WIF) and Github Actions.



STEP 1. Here are the products that are required to build the pipelines

GitHub Actions for CI/CD

IAM Workload Identity Federation (WIF) for secure, keyless authentication

Google Cloud Run for deploying your real-time app

Artifact Registry for storing container images


STEP 2. Create the Artifact Registry using the below command

gcloud artifacts repositories create my-docker-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Docker images repository"

us-central1-docker.pkg.dev/cts13-yemireddyc/documetn-github-actions


STEP 3. Create a Google Cloud Service Account and add the required roles.

# TODO: replace ${PROJECT_ID} with your value below.

gcloud iam service-accounts create "my-service-account" \
  --project "${PROJECT_ID}"

Please add the below roles:

i. Cloud Run Service Account Roles

Cloud Run Invoker (roles/run.invoker): Allows the service to be invoked.

Artifact Registry Reader (roles/artifactregistry.reader): Grants read access to pull container images from Artifact Registry.

Cloud Run Admin (roles/run.admin): Provides permissions to deploy and manage Cloud Run services.

Service Account User (roles/iam.serviceAccountUser): Grants permission to impersonate a service account.

ii. Cloud Build Service Account Roles

Cloud Build Editor (roles/cloudbuild.builds.editor): Allows Cloud Build to create and manage builds.

Artifact Registry Writer (roles/artifactregistry.writer): Grants permissions to push images to Artifact Registry.

Artifact Registry Reader (roles/artifactregistry.reader): Allows Cloud Build to pull base images from Artifact Registry.

iii. Workload Identity Federation (WIF) Configuration

Workload Identity Pool User (roles/iam.workloadIdentityUser): Allows the external identity to impersonate the Google Cloud service account.

Additionally, ensure that the external identity has the necessary permissions to interact with Cloud Run and Artifact Registry, as outlined above


STEP 4. Create a Workload Identity Pool:

# TODO: replace ${PROJECT_ID} with your value below.

gcloud iam workload-identity-pools create "github" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --display-name="GitHub Actions Pool"


STEP 5. Get the full ID of the Workload Identity Pool:

# TODO: replace ${PROJECT_ID} with your value below.

gcloud iam workload-identity-pools describe "github" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --format="value(name)"
  
This value should be of the format:
projects/123456789/locations/global/workloadIdentityPools/github


STEP 6. Create a Workload Identity Provider in that pool:

# TODO: replace ${PROJECT_ID} and ${GITHUB_ORG} with your values below.

gcloud iam workload-identity-pools providers create-oidc "my-repo" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --workload-identity-pool="github" \
  --display-name="My GitHub repo Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository,attribute.repository_owner=assertion.repository_owner" \
  --attribute-condition="assertion.repository_owner == '${GITHUB_ORG}'" \
  --issuer-uri="https://token.actions.githubusercontent.com"


STEP 7. Allow authentications from the Workload Identity Pool to your Google Cloud Service Account.

# TODO: replace ${PROJECT_ID}, ${WORKLOAD_IDENTITY_POOL_ID}, and ${REPO}
# with your values below.
#
# ${REPO} is the full repo name including the parent GitHub organization,
# such as "my-org/my-repo".
#
# ${WORKLOAD_IDENTITY_POOL_ID} is the full pool id, such as
# "projects/123456789/locations/global/workloadIdentityPools/github".

gcloud iam service-accounts add-iam-policy-binding "my-service-account@${PROJECT_ID}.iam.gserviceaccount.com" \
  --project="${PROJECT_ID}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${WORKLOAD_IDENTITY_POOL_ID}/attribute.repository/${REPO}"


STEP 8. Extract the Workload Identity Provider resource name:

# TODO: replace ${PROJECT_ID} with your value below.

gcloud iam workload-identity-pools providers describe "my-repo" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --workload-identity-pool="github" \
  --format="value(name)"
  
Use this value as the workload_identity_provider value in the GitHub Actions YAML:

- uses: 'google-github-actions/auth@v2'
  with:
    service_account: '...' # my-service-account@my-project.iam.gserviceaccount.com
    workload_identity_provider: '...' # "projects/123456789/locations/global/workloadIdentityPools/github/providers/my-repo"

STEP 9: Add the below variables and it's value in your github repository secrects and variables

PROJECT_ID
SERVICE
WORKLOAD_IDENTITY_PROVIDER

STEP 10: Add the below files to your Github Repo

File 1. main.py file

===

from flask import Flask

app = Flask(__name__)

@app.route('/')
def main():
        return 'Welcom to python flask world from github:cloudrun repo'
if __name__ == '__main__':
                  app.run(host='0.0.0.0', port=8080)

===


File 2. requirements.txt: Please mention all the required libraries in this file

===

Flask==3.0.3  

===

File 3. Dockerfile: Used to build the docker image 

===

FROM python:3.7-slim
RUN pip install flask
WORKDIR /myapp
COPY main.py /myapp/main.py
CMD ["python", "/myapp/main.py"]

File 4. Workflow yaml file: Create this file in the “.github/workflows” folder with “.yaml” Extension

name: 'Build and Deploy to Cloud Run'

on:
  push:
    branches:
      - '*'
env:
  PROJECT_ID:  ${{ vars.PROJECT_ID }} # TODO: update to your Google Cloud project ID
  REGION: 'us-central1' # TODO: update to your region
  SERVICE: ${{ vars.SERVICE }} # TODO: update to your service name
  WORKLOAD_IDENTITY_PROVIDER: ${{ vars.WORKLOAD_IDENTITY_PROVIDER }} # TODO: update to your workload identity provider
  REPO: 'documetn-github-actions' # Name of th repository in the Artifact Registry
  CRNAME: 'docgithubcr' # The Cloud Service deploy with the provided name  
  
jobs:
  deploy:
    runs-on: 'ubuntu-latest'
    permissions:
      contents: 'read'
      id-token: 'write'
    steps:
    - uses: 'actions/checkout@v3'

    # Configure Workload Identity Federation via a credentials file.
    - id: 'auth'
      name: 'Authenticate to Google Cloud'
      uses: 'google-github-actions/auth@v1'
      with:
        token_format: 'access_token'
        workload_identity_provider: ${{ vars.WORKLOAD_IDENTITY_PROVIDER }}
        service_account: ${{ vars.SERVICE }}
        
    - name: Configure Docker for Google Artifact Registry
      run: |-
          gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev --quiet

    - name: 'Build and Push Container'
      run: |-
          DOCKER_TAG="${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPO }}/myfimage:${{ github.sha }}"
          docker build --tag "${DOCKER_TAG}" .
          docker push "${DOCKER_TAG}"
    - name: 'Deploy to Cloud Run'

        # END - Docker auth and build

      uses: 'google-github-actions/deploy-cloudrun@33553064113a37d688aa6937bacbdc481580be17' # google-github-actions/deploy-cloudrun@v2
      with:
          service: '${{ env.CRNAME }}'
          region: '${{ env.REGION }}'
          # NOTE: If using a pre-built image, update the image name below:
          image: '${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPO }}/myfimage:${{ github.sha }}'
      # If required, use the Cloud Run URL output in later steps
    - name: 'Show output'
      run: |2-

          echo ${{ steps.deploy.outputs.url }}
         
===

STEP 10:

Go to th Actions of the github repo and monitor the pipeline



        

  



