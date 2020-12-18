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

### Redis

**There is no free tier for Memorystore for Redis.** For tips to keep costs low during development, see configuration suggestions [below](#saving-money-on-development-deployments).

If you plan to run Redis, you will need to go through some extra configuration:
1. Enable the Memorystore API in your project.
1. Create a Memorystore for Redis instance.
    1. Make note of the Authorized VPC Network you configure. The default network will be adequate for a simple use case.
    1. Once the instance is created, make note of the IP and port.
1. Enable the Serverless VPC Access API in your project.
1. Go to Serverless VPC Access in the console and create a connector.
    1. Choose the same network that your Redis instance is configured on.
    1. Choose an IP range. The example `10.8.0.0/28` will probably work fine.

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
       1. If you are using Redis, set `REDIS_URL` to `redis://<IP for your instance>:<port for your instance>` (see the [Redis HOWTO](HOWTO_CONNECT_WITH_REDIS.md) for more options to add here).
    1. Under "Connections", click the "Add Connection" button for "Cloud SQL Connections" and select your Cloud SQL instance in the dropdown.
        1. If you are using Redis, select the VPC connector you created earlier under "VPC Connector" and choose the "Route only requests to private IPs through the VPC connector" option.
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

If you are experimenting with a non-production deployment for personal testing, you should be aware that many of the GCP resources you need will incur charges. Here are some tips to keep your costs low.

### Cloud Run, Cloud Storage

If you have a small development instance that you are only using for manual testing, you are very likely to stay within [free tier](https://cloud.google.com/free/docs/gcp-free-tier#free-tier) usage limits for Cloud Run and Cloud Storage (used for artifacts from Run and Build), as long as you are operating in the `us-central1`, `us-east1`, or `us-west1` region. You can reduce Cloud Storage usage if necessary by deleting old images in container registry and deleting old revisions of your Cloud Run service.

### Cloud Build

You might exceed free tier limits if you build many container images (>20 or so) per day. If you're concerned about this, you can [build images locally](https://cloud.google.com/run/docs/building/containers#docker) and push them to container registry manually without too much additional work.

### Cloud SQL

You will be charged for Cloud SQL. As of 14 Dec 2020, the least expensive Cloud SQL Postgres instance costs about 9 USD per month, if configured as follows:
1. Under "Machine type and storage": set machine cores to 1 shared vCPU, memory to 0.6 GB, storage type to HDD, storage capacity to 10 GB, and disable automatic storage increases.
1. Under "Backups, recovery, and high availability": disable automatic backups and set availability to "single zone."

Such an instance is probably adequate for use during development. You can significantly reduce costs during development by [stopping your Cloud SQL instance](https://cloud.google.com/sql/docs/postgres/start-stop-restart-instance) when you are not actively using it and only starting it when you need it. Note that even a stopped instance will still incur some charges (charges for storage will continue), but this can drop the cost to ~2 USD per month if you only run the instance for a couple of hours each day. You can see full pricing information https://cloud.google.com/sql/pricing#pg-pricing.

### Memorystore for Redis

Redis instances can quickly add up in cost -- the least expensive instance (as of 18 Dec 2020) costs about 35 USD per month, if configured in the Basic tier at the smallest size (1 GB).

Unfortunately you cannot stop a Redis instance as with Cloud SQL. If you are concerned about surprises on your credit card bill from an development deployment, you might want to delete your Redis instance when not actively working with it. Unfortunately this is a bit of a pain, since, deleting and recreating Redis instances will change the IP, so you'll need to adjust the configuration of your Spoke service each time.

## TODO

In no particular order:
* Figure out minimal IAM permissions
* ~~Auth0~~
    * Just follow standard docs and it works
* Custom domains
    * Works, needs some GCP-specific documentation (although standard GCP docs are straightforward enough)
* ~~Redis~~
    * Documented, maybe add static IPs
* Updating
* See if you can do a free tier deployment by DIYing SQL instead of using Cloud SQL
* ~~Verify that calls to external APIs (e.g. Twilio, Mailgun) work~~
    * Verified Twilio, standard docs work fine. I have to imagine Mailgun is the same.
* Export to GCS
* Multihoming?
* Provide a YAML configuration for Cloud Run
* Terraform?
* [Berglas?](https://github.com/GoogleCloudPlatform/berglas)
