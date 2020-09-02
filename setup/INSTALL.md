# Installation Guide

## Getting Started

### Requirements
1.  Install [GCloud SDK](https://cloud.google.com/sdk/install).

1.  You will also need to be owner on a GCP project that has billing enabled.  You may either use this project to house the architecture for Four Keys, or you will be given the option to create new projects.  If you create new projects, the original project will NOT be altered during set up, but the billing information from this parent project will be applied to any projects created.

Once you have your parent project, run the following from the top-level directory of this repository:
```bash
gcloud config set project $PARENT_PROJECT_ID
cd setup
./setup.sh 2>&1 | tee setup.log
```

### Manual Prompts
The setup script will prompt you for the following input:

- Would you like to create a new Google Cloud Project for the four key metrics? (y/n)
  - If you choose no, you will be asked to input and confirm the ID of the project that you want to use.
- Are you using Gitlab? (y/n)
  - If you choose yes, the GitLab specific Pub/Sub topic, subscriptions, and worker will be created.
- Are you using Github? (y/n)
  - If you choose yes, the GitHub specific Pub/Sub topic, subscriptions, and worker will be created.
- BigQuery setup
  - If you've never setup BigQuery before, a setup page will open in your browser.
- Would you like to create a separate new project to test deployments for the four key metrics? (y/n)
  - You have the option of creating a new project to test out doing deployments and seeing how they are tracked in the dashboard.  However, if you already have a project with deployments, you may select no to skip this step.  You do not need to select yes to generate mock data.
- Would you like to generate mock data? (y/n)
  - If you select yes, a script will run through and send mock GitLab or GitHub events to your event-handler.  This will populate your dashboard with mock data.  The mock data will include the work "mock" in the source.

### New Projects
If you've chosen to create new projects, after the script finishes you will have two new projects named in the env.sh file of the form `fourkeys-XXXX` and `helloworld-XXXXX`.  The `fourkeys-XXXX` project will be home to all the services that collect data from your deployments, while `helloworld-XXXX` will be the staging and prod deployments for your application.

Later if you want to remove the newly created projects and all associated data, you can run the `cleanup.sh`.  **Only do this when you are done experimenting with fourkeys entirely, or want to start over because `cleanup.sh` will remove the projects and all the collected data.**

## The Setup Explained
The setup script is going to do many things to help create the service architecture described in the `README.md`.  The script will output the commands you would need to do manually.

The steps are:
- Create randomly generated project names
- Save project names in env.sh
- Set up Four Keys Project
  - Create project
  - Link billing to parent project
  - Enable Apis
  - Add IAM Policy Bindings
  - Create PubSub Topics
  - Deploy Event Handler
  - Deploy BigQuery GitHub and/or GitLab Worker
  - Deploy BigQuery Cloud Build Worker
  - Create BigQuery PubSub Subscriptions
  - Create BigQuery Dataset, Tables, and Scheduled Queries
- Set up Helloworld Project
  - Create Project
  - Link billing to parent project
  - Enable Apis
  - Deploy Helloworld to staging
  - Deploy Helloworld to prod
- Generate mock data using the scripts found in the data_generator/ directory
- Connect to the DataStudio Dashboard template
  - Select organization and project
  - Click "Create Report" on the next screen with the list of fields


## Integrate with a live repo

The setup script can create mock data, but it cannot integrate automatically with our live projects.  To measure our team's performance, we should hook up the services to a live repo with ongoing deployments so we can experiment with how changes, successful deployments, and failed deployments affect our statistics.

Integrating with a live repo will require a few extra manual steps:

### Collect changes data

#### GitHub instructions
1.  Navigate to your GitHub repo
    - If you're using the `Helloworld` sample, fork the demo by navigating to the [GitHub repo](https://github.com/knative/docs.git) and clicking **Fork**.
1.  From your repo (or forked repo), click **Settings**.
1.  Select **Webhooks** from the left hand side.
1.  Click **Add Webhook**
1.  Get the Event Handler endpoint for your Four Keys service:
    ```bash
    . ./env.sh
    gcloud config set project ${FOURKEYS_PROJECT}
    gcloud run --platform managed --region ${FOURKEYS_REGION} services describe event-handler --format=yaml | grep url | head -1 | sed -e 's/  *url: //g'
    ```
1.  In the **Add Webhook** interface, use the Event Handler endpoint for **Payload URL**
1.  Run the following command to get the secret from Google Secrets Manager:
    ```bash
    gcloud secrets versions access 1 --secret="github-secret"
    ```
1.  Put the secret in the box labelled **Secret**
1.  Select 'application/json' for **Content Type**
1.  Select **Send me everything**
1.  Click **Add Webhook**

#### GitLab instructions
1.  Navigate to your repo and click **Settings**.
1.  Select `Webhooks` from the menu.
1.  Get the Event Handler endpoint for your Four Keys service:
    ```bash
    . ./env.sh
    gcloud config set project ${FOURKEYS_PROJECT}
    gcloud run --platform managed --region ${FOURKEYS_REGION} services describe event-handler --format=yaml | grep url | head -1 | sed -e 's/  *url: //g'
    ```
1.  Use the Event Handler endpoint for **Payload URL**.
1.  Run the following command to get the secret from Google Secrets Manager:
    ```bash
    gcloud secrets versions access 1 --secret="github-secret"
    ```
1.  Put the secret in the box labelled **Secret Token**.
1.  Select all of the checkboxes.
1.  Leave the `Enable SSL verification` selected.
1.  Click **Add Webhook**.

### Collect deployment data

#### Configuring Cloud Build to deploy on GitHub Pull Request merges
1.  Go back to your repo's main page.
1.  At the top of the GitHub page, click **Marketplace**.
1.  Search for **Cloud Build**.
1.  Select **Google Cloud Build**.
1.  Click **Set Up Plan**.
1.  Click **Set up with Google Cloud Build**.
1.  Select **Only select repositories**.
1.  Fill in your forked repo.
1.  Log in to Google Cloud Platform.
1.  Add your new FourKeys project named `fourkeys-XXXXX`.
1.  Select your repo.
1.  Click **Connect repository**.
1.  Click **Create push trigger**.

And now, whenever a pull request is merged into master of your fork, Cloud Build will trigger a deploy into prod and data will flow into your Four Keys project.

#### Configuring Cloud Build to deploy on Gitlab merges
1.  Go to your fourkeys project and [create a service account](https://cloud.google.com/iam/docs/creating-managing-service-accounts#iam-service-accounts-create-console) called `gitlab-deploy`.
1.  [Create a JSON service account key](https://cloud.google.com/iam/docs/creating-managing-service-account-keys#iam-service-account-keys-create-console) for your `gitlab-deploy` service account.
1.  In your GitLab Repo, navigate to **Settings** on the left-hand menu and then select **CI/CD**
1.  Save your account key under variables.
    1.  Input SERVICE_ACCOUNT in the **key** field.
    1.  Input the JSON in the **Value** field. 
    1.  Select **Protect variable**.
1.  Save your Google Cloud project ID under variables.
    1.  Input `PROJECT_ID` in the **key** field.
    1.  Input your project-id in the **value** field.
1.  Add a `.gitlab-ci.yml` file to your repo:
    ```
    image: google/cloud-sdk:alpine

    deploy_production:
      stage: deploy
      environment: Production
      only:
      - master
      script:
      - echo $SERVICE_ACCOUNT > /tmp/$CI_PIPELINE_ID.json
      - gcloud auth activate-service-account --key-file /tmp/$CI_PIPELINE_ID.json
      - gcloud builds submit . --project $PROJECT_ID
      after_script:
      - rm /tmp/$CI_PIPELINE_ID.json
    ```

This setup will trigger a deployment on any `push` to the `master` branch.

### Collect incident data

For this demo, we're using GitLab and/or GitHub issues to track incidents.  

#### Create an incident

1.  Open an issue
1.  Add the tag `Incident`
1.  In the body of the issue, input `root cause: {SHA of the commit}`

When the incident is resolved, close the issue. The incident will be measured from the time of the deployment to the resolution of the issue.  
