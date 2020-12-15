_Work in progress_

# Deploying to Google Cloud Run

Quick sketch of a basic deployment of Spoke to Google Cloud Run using Cloud SQL as the backing database. This is not production ready.

## Prerequisites

You can either [install the Google Cloud SDK](https://cloud.google.com/sdk/docs/install) on your local machine or run these commands in a [Cloud Shell](https://cloud.google.com/shell).

## Prepare GCP resources

1. Select or [create](https://cloud.google.com/resource-manager/docs/creating-managing-projects#creating_a_project) a Google Cloud project. The project needs to be linked to a billing account. Make note of your project name.
1. [Enable the following APIs](https://cloud.google.com/service-usage/docs/enable-disable) in your project: Cloud Run, Cloud SQL, Cloud SQL Admin, Cloud Build.
1. [Create a Cloud SQL Postgres instance.](https://cloud.google.com/sql/docs/postgres/create-instance)
    1. Make sure to enable a public IP.
    1. Make note of the database password that you set -- you will need this when deploying Spoke.
    1. **There is no free tier for Cloud SQL.** For tips to keep costs low during development, see configuration suggestions [below](#saving-money-on-development-deployments).
    1. View the newly created SQL instance in the Cloud Console and note its [_Connection name_](https://cloud.google.com/sql/docs/postgres/instance-info#connect_to_this_instance). This will have the form `<project name>:<location>:<sql instance name>`.
1. [Build and push](https://cloud.google.com/run/docs/building/containers) a Spoke container image to the container registry.
    1. The easiest way to do this is via Cloud Build from the root directory of your Spoke repository:
    ```shell
    gcloud builds submit --tag gcr.io/<your cloud project name>/spoke
    ```

## Deploy to Cloud Run

You can now [deploy the Spoke container image to Cloud Run](https://cloud.google.com/run/docs/deploying#service) with the following settings:
1. Select "Cloud Run (fully managed)" as the deployment platform.
1. On the second page of the service creation form, in the "Container image URL" box, enter the image URL that you used when building the image using Cloud Build, e.g., `gcr.io/<your cloud project name>/spoke`.
1. Click "Show Advanced Settings" to set the following:
    1. Set at least the following environment variables:

    | Name | Value |
    | ---- | ----- |
    | `PASSPORT_STRATEGY` | `local` |
    | `JOBS_SAME_PROCESS` | `1` |
    | `DB_TYPE` | `pg` |
    | `DB_USER` | `postgres` |
    | `DB_PASSWORD` | the Cloud SQL password you set earlier |
    | `SESSION_SECRET` | random key |
    | `DB_SOCKET_PATH` | `/cloudsql` |
    | `CLOUD_SQL_CONNECTION_NAME` | the Cloud SQL connection name you noted earlier |
    | `KNEX_MIGRATION_DIR` | `/spoke/build/server/migrations/` |

    You can also set any other environment variables that are specific to your configuration.
    1. Under "Connections", click the "Add Connection" button for "Cloud SQL Connections" and select your Cloud SQL instance in the dropdown.
1. Under "Authentication", select "Allow unauthenticated invocations".
1. Create the instance.
1. After the instance is running, go to the Cloud Run section of the [Cloud Console](https://console.cloud.google.com) and view the details for your new service. Note the URL shown at the header (something like `https://<service name>-xxxxxxxx.a.run.app`). This is the URL you will visit to use Spoke. We also need to set the `BASE_URL` environment variable to that value:
    1. Click the "Edit & Deploy New Revision" button.
    1. Go to the "Variables" tab under "Advanced settings".
    1. Add a variable with name `BASE_URL` and value the URL you copied earlier.
    1. Check "Serve this revision immediately".
    1. Click "Deploy".

Your Spoke instance should now be running! On your first run, it can take several minutes for the initial database tables to be created, and it will seem like Spoke is unresponsive until the tables are up. You can look at the logs for your Cloud Run service get some clues if that is the case: if you see an entry saying `CREATING DATABASE SCHEMA` and then errors of the form `relation "xxxxx" does not exist`, that probably means the schema is not fully initialized yet.

## Saving money on development deployments

If you have a small development instance that you are only using for manual testing, you are very likely to stay within [free tier](https://cloud.google.com/free/docs/gcp-free-tier#free-tier) usage limits for Cloud Run, Cloud Build, and Cloud Storage (used for artifacts from Run and Build), as long as you are operating in the `us-central1`, `us-east1`, or `us-west1` region. You will be charged for Cloud SQL. As of 14 Dec 2020, the least expensive Cloud SQL Postgres instance costs about 9 USD per month, if configured as follows:
1. Under "Machine type and storage": set machine cores to 1 shared vCPU, memory to 0.6 GB, storage type to HDD, storage capacity to 10 GB, and disable automatic storage increases.
1. Under "Backups, recovery, and high availability": disable automatic backups and set availability to "single zone."

Such an instance is probably adequate for use during development. You can significantly reduce costs during development by [stopping your Cloud SQL instance](https://cloud.google.com/sql/docs/postgres/start-stop-restart-instance) when you are not actively using it and only starting it when you need it. Note that even a stopped instance will still incur some charges (charges for storage will continue), but this can drop the cost to ~2 USD per month if you only run the instance for a couple of hours each day. You can see full pricing information https://cloud.google.com/sql/pricing#pg-pricing.

## TODO

In no particular order:
* Figure out minimal IAM permissions
* Auth0
* Custom domains
* Redis
* Updating
* See if you can do a free tier deployment by DIYing SQL instead of using Cloud SQL
* Verify that calls to external APIs (e.g. Twilio, Mailgun) work
* Multihoming?
* Provide a YAML configuration for Cloud Run
* Terraform?
