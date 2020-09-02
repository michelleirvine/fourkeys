# Four Keys


# Background

Through six years of research, the [DevOps Research and Assessment (DORA)](https://cloud.google.com/blog/products/devops-sre/the-2019-accelerate-state-of-devops-elite-performance-productivity-and-scaling) team has identified four key metrics that indicate the performance of a software development team.  This Four Keys project allows you to collect data from your GitHub or GitLab repos. It then aggregates the data, compiles it into a dashboard displaying these key metrics, and color-codes each metric.

The four key metrics are:

*   **Deployment Frequency**
*   **Lead Time for Changes**
*   **Time to Restore Services**
*   **Change Failure Rate**

# Who should use the Four Keys project

This Four Keys project is useful for anyone who wants to view and track the software development performance of a project in GitHub or GitLab. Perhaps you want to measure the impact of changes you've rolled out with your team, such as new tooling or more automated test coverage. Or maybe you want to know the baseline performance of your projects. Four Keys allows you to monitor performance by tracking these four key metrics.

For in a quick measurement of your software delivery performance, and suggestion about how to improve, take the [quick check](https://www.devops-research.com/quickcheck.html).

If you're working to improve your teams' software delivery performance, Four Keys will help you measure your progress. Using Four Keys can also help you with specific capabilities that the [State of DevOps Report](https://cloud.google.com/devops) has shown to drive higher software delivery performance. These capabilities are:

*   [Monitoring and observability](https://cloud.google.com/solutions/devops/devops-measurement-monitoring-and-observability)
*   [Visual management capabilities](https://cloud.google.com/solutions/devops/devops-measurement-visual-management)
*   [Monitoring systems to inform business decisions](https://cloud.google.com/solutions/devops/devops-measurement-monitoring-systems)

Four Keys is useful for projects that have deployments. Projects with releases have data that doesn't map as well in the Four Keys project.

# How it works


1.  Events from your project are sent to a webhook target hosted on Cloud Run. Events are occurances in you system (such as GitHub) that you can measure. A pull request or filed bug/issue are examples of events.
1.  The Cloud Run target publishes all events to Pub/Sub.
1.  A Cloud Run instance is subscribed to the Pub/Sub topics. The instance does some light data transformation and inputs the data into BigQuery.
1.  Nightly scripts are scheduled in BigQuery to complete the data transformations and feed into the dashboard.

This diagram shows the design of thte Four Keys project:

![Diagram showing the Four Keys design](images/fourkeys-design.png)


# Code structure


* `bq_workers/`
  * Contains the code for the individual BigQuery workers.  Each data source has its own worker service with the logic for parsing the data from the pub/sub message. For example, GitHub has its own worker which only looks at events pushed to the GitHub-Hookshot pub/sub topic.
* `data_generator/`
  * Contains a python script for generating mock GitHub data.
* `event_handler/`
  * Contains the code for the `event_handler`. This is the public service that accepts incoming webhooks.  
* `queries/`
  * Contains the SQL queries for creating the derived tables.
  * Contains a python script for schedulig the queries.
* `setup/`
  * Contains the code for setting up and tearing down the fourkeys pipeline. Also contains a script for extending the data sources.
* `shared/`
  * Contains a shared module for inserting data into BigQuery, which is used by the `bq_workers`.


# How to use 


## Out of the box

_The project uses Python 3 and supports data extraction for Cloud Build and GitHub events._



1.  Fork this project.
1.  Run the automation scripts, which do following (See the [INSTALL.md](setup/INSTALL.md) for more details):
    1.  Set up a new Google Cloud Project.
    1.  Create and deploy the Cloud Run webhook target and ETL workers.
    1.  Create the Pub/Sub topics and subscriptions.
    1.  Enable the Google Secret Manager and create a secret for your GitHub repo.
    1.  Create a BigQuery dataset and tables, and schedule the nightly scripts.
    1.  Open up a browser tab to connect your data to a DataStudio dashboard template.
1.  Set up your development environment to send events to the webhook created in the second step:
    1.  Add the secret to your GitHub webhook.


## Generate mock data

You can generate mock data to try out and become familiar with the Four Keys project.

When you install Four Keys, the setup script has an option to generate mock data. To generate data, select "yes."

The data generator creates mocked GitHub events, which is ingested into the table with the source “githubmock.”   It creates following events: 

* 5 mock commits with timestamps no earlier than a week ago
  * _Note: Number can be adjusted_
* 1 associated deployment
* Associated mock incidents 
  * _Note: By default, less than 15% of deployments create a mock incident. Threshold can be adjusted in the script._

To generate mock data outside of the setup script, ensure that you’ve saved your webhook URL and GitHub Secret in your environment variables:

```sh
export WEBHOOK={your event handler URL}
export GITHUB_SECRET={your GitHub signing secret}
```

Then run the following command:

```sh
python3 data_generator/data.py
```

These events then run through the pipeline:
*  The event handler logs will show successful requests.
*  The PubSub topic will show messages posted.
*  The BigQuery GitHub parser will show successful requests.
*  You can query the `events_raw` table directly in BigQuery using the following:


```sql
SELECT * FROM four_keys.events_raw WHERE source = 'githubmock';
```


## Reclassify events and update your queries

The scripts consider some events to be “changes,” “deploys,” and “incidents.”   You can reclassify one of the events in the table. For example, you may want to use a label for your incidents other than “incident.” To reclassify an event, update the nightly scripts in BigQuery for the following tables:



*   `four\_keys.changes`
*   `four\_keys.deployments`
*   `four\_keys.incidents`

To update the scripts:

1.  Update the `sql` files in the `queries` folder, rather than in the BigQuery UI.
1.  Once you've edited the SQL, run the `schedule.py` script to update the scheduled query that populates the table.  For example, if you wanted to update the `four_keys.changes` table, you'd run:

```sh 
python3 schedule.py --query_file=changes.sql --table=changes
```

Notes: 

*   The `query_file` flag should contain the relative path of the file.  
*   To feed into the dashboard, the table name should be: `changes`, `deployments`, or `incidents`. 


## Extend to other event sources

The Four Keys project has some initial events defined. You can add other events.

1.  Add to the `AUTHORIZED_SOURCES` in `sources.py`
    1.  If you want to create a verification function, add the function to the file as well.
1.  Run the `new_source.sh` script in the `setup` directory. This script will create a PubSub topic, a PubSub subscription, and the new service using the `new_source_template`.
    1.  Update the `main.py` in the new service to parse the data properly.
1.  Update the BigQuery Script to classify the data properly.

**If you add a common data source as a new event, please submit a pull request to add this event to the Four Keys project so that others may benefit from the functionality.**


## Run tests
This project uses nox to manage tests.  Ensure that nox is installed:

```sh
pip install nox
```

The noxfile defines what tests will run on the project.  It’s set up to run all the `pytest` files in all the directories, as well as run a linter on all directories.   To run nox:

```sh
python3 -m nox
```

### List tests

To list all the test sesssions in the noxfile:

```sh
python3 -m nox -l
```

### Run a specific test

Once you have the list of test sessions, you can run a specific session with:

```sh
python3 -m nox -s "{name_of_session}" 
```

The `name_of_session` is something like `py-3.6(folder='.....')`.  

# Data Schema


### four\_keys.events\_raw


<table>
  <tr>
   <td><strong>Field Name</strong>
   </td>
   <td><strong>Type</strong>
   </td>
   <td><strong>Notes</strong>
   </td>
  </tr>
  <tr>
   <td>`source`
   </td>
   <td>strings
   </td>
   <td>For example, GitHub
   </td>
  </tr>
  <tr>
   <td>`event_type`
   </td>
   <td>strings
   </td>
   <td>For example, push
   </td>
  </tr>
  <tr>
   <td>🔑`id`
   </td>
   <td>strings
   </td>
   <td>ID of the development object. For example, bug ID, commit ID, Pull Request ID.
   </td>
  </tr>
  <tr>
   <td>`metadata`
   </td>
   <td>JSON
   </td>
   <td>Body of the event
   </td>
  </tr>
  <tr>
   <td>`time_created`
   </td>
   <td>timestamp
   </td>
   <td>The time the event was created
   </td>
  </tr>
  <tr>
   <td>`signature`
   </td>
   <td>strings
   </td>
   <td>Encrypted signature key from the event. This is the <strong>unique key</strong> for the table.  
   </td>
  </tr>
  <tr>
   <td>`msg_id`
   </td>
   <td>strings
   </td>
   <td>Message ID from Pub/Sub
   </td>
  </tr>
</table>
The key icon (🔑) indicates this ID is generated by the original system, such as GitHub.

This table is used to create the following three derived tables: 


#### four\_keys.deployments 

_Note: Deployments and changes have a many to one relationship.  Table only contains successful deployments._


<table>
  <tr>
   <td><strong>Field Name</strong>
   </td>
   <td><strong>Type</strong>
   </td>
   <td><strong>Notes</strong>
   </td>
  </tr>
  <tr>
   <td>🔑`deploy_id`
   </td>
   <td>string
   </td>
   <td>ID of the deployment - foreign key to ID in `events_raw`
   </td>
  </tr>
  <tr>
   <td>`changes`
   </td>
   <td>array of strings
   </td>
   <td>List of IDs associated with the deployment. For example, commit_ids, bug_ids.  
   </td>
  </tr>
  <tr>
   <td>`time_created`
   </td>
   <td>timestamp
   </td>
   <td>Time the deployment was completed
   </td>
  </tr>
</table>
The key icon (🔑) indicates this ID is generated by the original system, such as GitHub.


#### four\_keys.changes


<table>
  <tr>
   <td><strong>Field Name</strong>
   </td>
   <td><strong>Type</strong>
   </td>
   <td><strong>Notes</strong>
   </td>
  </tr>
  <tr>
   <td>🔑`change_id`
   </td>
   <td>string
   </td>
   <td>ID of the change - foreign key to ID in events_raw
   </td>
  </tr>
  <tr>
   <td>`time_created`
   </td>
   <td>timestamp
   </td>
   <td>Time_created from events_raw
   </td>
  </tr>
  <tr>
   <td>`change_type`
   </td>
   <td>string
   </td>
   <td>The event type
   </td>
  </tr>
</table>
The key icon (🔑) indicates this ID is generated by the original system, such as GitHub.


#### four\_keys.incidents


<table>
  <tr>
   <td><strong>Field Name</strong>
   </td>
   <td><strong>Type</strong>
   </td>
   <td><strong>Notes</strong>
   </td>
  </tr>
  <tr>
   <td>🔑`incident_id`
   </td>
   <td>string
   </td>
   <td>Id of the failure incident
   </td>
  </tr>
  <tr>
   <td>`changes`
   </td>
   <td>array of strings
   </td>
   <td>List of deployment IDs that caused the failure
   </td>
  </tr>
  <tr>
   <td>`time_created`
   </td>
   <td>timestamp
   </td>
   <td>Min timestamp from changes
   </td>
  </tr>
  <tr>
   <td>`time_resolved`
   </td>
   <td>timestamp
   </td>
   <td>Time the incident was resolved
   </td>
  </tr>
</table>
The key icon (🔑) indicates this ID is generated by the original system, such as GitHub.


# Dashboard 

The dashboard displays all four metrics with daily systems data, as well as a current snapshot of the last 90 days.  

To understand the metrics and intent of the dashboard, see the [2019 State of DevOps Report](https://cloud.google.com/devops).


## Metrics Definitions

**Deployment Frequency**



*   The number of deployments per time period: daily, weekly, monthly, yearly. 

**Lead Time for Changes**



*   The median amount of time for a commit to be deployed into production.

**Time to Restore Services**



*   For a failure, the median amount of time between the deployment which caused the failure, and the restoration.  The restoration is measured by closing an associated bug / issue / incident report. 

**Change Failure Rate**



*   The number of failures per the number of deployments. For example, if there are four deployments in a day and one causes a failure, that is a 25% change failure rate.


## Color Coding

The color coding of the quarterly snapshots roughly follows the buckets determined by the [State of DevOps Report](https://cloud.google.com/devops).  

**Deployment Frequency**



*   **Green:** Weekly
*   **Yellow:** Monthly
*   **Red:** Between once per month and once every 6 months  
    *   This is expressed as “Yearly” 

**Lead Time to Change**



*   **Green:** Less than one week
*   **Yellow:** Between one week and one month
*   **Red:** Between one month and 6 months  
*   **Red:** Anything greater than 6 months
    *   This is expressed as “One year” 

**Time to Restore Service**



*   **Green:** Less than one day
*   **Yellow:** Less than one week
*   **Red:**  Between one week and a month
    *   This is expressed as “One month” 
*   **Red:** Anything greater than a month
    *   This is expressed as “One year” 

**Change Failure Rate**



*   **Green:** Less than 15%
*   **Yellow:** 16% - 45%
*   **Red:**  Anything greater than 45%

The following chart is from the [State of DevOps Report](https://cloud.google.com/devops). It describes the range of the key metrics for teams that have elite, high, medium, or low software delivery performance.

![From the State of DevOps Report, chart describing each key metric's range for different software delivery performance levels.](images/dora-chart.png)

Disclaimer: This is not an officially supported Google product
