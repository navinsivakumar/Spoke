# Configuring data exports
This guide explains how Spoke manages data exports and how to configure them in your installation.

## Conceptual overview
When a user requests a data export, Spoke prepares it behind the scenes. When the data is ready, it is uploaded to a cloud object store (currently either an Amazon Web Services (AWS) S3 bucket or Google Cloud Storage (GCS) bucket). A bucket is a cloud storage container. Once the exported data has been added to the bucket, Spoke sends an email notification to the user who requested the export with a link to download the data file.

To enable data exporting, you will need:
1. to configure Spoke to send emails,
2. access to an AWS account or GCP service account, and
3. a bucket which that account can access.

## S3 vs. Bucketeer
If you have deployed Spoke to Heroku, you can use the [Bucketeer add-on](https://elements.heroku.com/addons/bucketeer) instead of configuring your own S3 bucket. Bucketeer automatically provisions S3 storage for you, enabling you to skip the numbered steps below. The tradeoff is that Bucketeer charges you a minimum of $5/month. Depending on your [usage](https://aws.amazon.com/s3/pricing/), this may well be more than you'd pay for your own S3 bucket, especially if you're new to AWS and taking advantage of the [free tier](https://aws.amazon.com/free/).

To use Bucketeer, [skip to the end of this document](#bucketeer-setup).

## S3 setup

__Skip this section__ if you are using Bucketeer.
  1. __[Configure Spoke to send emails](HOWTO_EMAIL_CONFIGURATION.md).__
  2. __Create an AWS account.__ If you already have an AWS account, skip this step. Otherwise, see [Amazon's documentation](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/) to create an AWS account.
  3. __Sign up for S3.__ If you already have S3, skip this step. Otherwise, see [Amazon's documentation](https://docs.aws.amazon.com/AmazonS3/latest/gsg/SigningUpforS3.html) to sign up for S3 using your AWS account.
  4. __Create a S3 bucket.__ If you already have an S3 bucket, skip this step. Otherwise, see [Amazon's documentation](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/create-bucket.html) to create an S3 bucket. You __don't__ need to enable public access to the bucket.
  5. __Configure Spoke environment variables.__ In order for Spoke to connect to S3, the following environment variables must be set:
    - `AWS_ACCESS_KEY_ID`
    - `AWS_S3_BUCKET_NAME`
    - `AWS_SECRET_ACCESS_KEY`

If you've reached this point in application setup, you've probably configured environment variables already. Here are [Heroku](https://devcenter.heroku.com/articles/config-vars#managing-config-vars) and [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/env_variables.html) instructions. Locally, you can use a `.env` file or the like.

## Bucketeer setup
__Skip this section__ if you have set up your own S3 bucket.

1. __Provision the Bucketeer add-on.__ Use the Heroku CLI (instructions [here](https://devcenter.heroku.com/articles/bucketeer#provisioning-the-add-on)) or the Heroku dashboard. You can [migrate](https://devcenter.heroku.com/articles/bucketeer#migrating-between-plans) between plans at any time with no downtime.
2. __Modify the default environment variables.__ Bucketeer sets the following environment variables:
  - `BUCKETEER_AWS_ACCESS_KEY_ID`
  - `BUCKETEER_AWS_SECRET_ACCESS_KEY`
  - `BUCKETEER_BUCKET_NAME`

Spoke, however, expects these names:
- `AWS_ACCESS_KEY_ID`
- `AWS_S3_BUCKET_NAME`
- `AWS_SECRET_ACCESS_KEY`

You can change the names in the [dashboard](https://devcenter.heroku.com/articles/config-vars#using-the-heroku-dashboard) or via the [CLI](https://devcenter.heroku.com/articles/config-vars#using-the-heroku-cli). `heroku config:edit` will open all environment variables in an interactive editor.

## GCS setup

__Note__: if you have configured S3 export above, then Spoke will export to S3 only and not GCS, even if you follow the configuration below. You must unset the S3 environment variables in order for GCS export to work.

1.  __[Configure Spoke to send emails](HOWTO_EMAIL_CONFIGURATION.md).__
1.  Select your Google Cloud project.
1.  Set up a service account:
    1.  If you do not have a service account already (e.g., if you are not running Spoke in a GCP environment like Cloud Run or GKE), create a service account. You do not need to set any IAM permissions for the new service account on the project. Note the service account name and email address.
    1.  If you already have a [Spoke service account](HOWTO_DEPLOYING_GOOGLE_CLOUD_RUN.md#service-account-configuration) (i.e. you are already running the Spoke service in a GCP environment like Cloud Run or GKE), make note of the service account name and email address for use in the following steps. You can follow this guide even if you are using the default service account for Spoke.
1.  Create a GCS bucket and make note of its name.
1.  Grant the IAM roles `roles/storage.objectCreator` and `roles/storage.objectViewer` to your service account on the bucket.
1.  Configure the Spoke service by choosing th:
    1.  If you are running in a GCP environment that can retrieve its own service account credentials (i.e. most GCP compute platforms, including Cloud Run, GKE, GCE, GAE, etc.):
        1.  Grant the IAM role `roles/iam.serviceAccountTokenCreator` to the Spoke service account on itself. This is necessary to allow Spoke service, which runs as the service account, to call the `signBlob` API to sign URLs with a server-side signing key for its own service account.
        1.  Configure environment variables:
        | Name | Value |
        | ---- | ----- |
        | `GCP_ACCESS_AVAILABLE` | `true` |
        | `GCP_STORAGE_BUCKET_NAME` | name of the bucket you created |
        | `GCLOUD_PROJECT` | if the Spoke service account and GCS bucket are in different projects, set this to the project ID which contains the GCS bucket. Otherwise leave unset |
  1.  If you do not have automatic access to service account credentials and you are running in an environment where you can add a secure volume mount (e.g. Kubernetes secrets in a non-GKE cluster):
      1.  [Create a JSON key](https://console.cloud.google.com/apis/credentials/serviceaccountkey) for your Spoke service account. A file containing the JSON key will download to your computer.
          1.  __Note:__ the JSON key is a private credential. Anyone holding the key can act as your Spoke service account, so keep it secure and *do not* check it into a source code repository! If the key gets into the wrong hands, you can disable it on your service account and generate a new key.
      1.  Make the file available at a volume mount in your service.
      1.  Configure environment variables:
      | Name | Value |
      | ---- | ----- |
      | `GOOGLE_APPLICATION_CREDENTIALS` | path to the file mount containing the JSON key |
      | `GCP_STORAGE_BUCKET_NAME` | name of the bucket you created |
      | `GCLOUD_PROJECT` | if the Spoke service account and GCS bucket are in different projects, set this to the project ID which contains the GCS bucket. Otherwise leave unset |
  1.  If you do not have access to configure volume mounts (e.g. Heroku):
      1.  [Create a JSON key](https://console.cloud.google.com/apis/credentials/serviceaccountkey) for your Spoke service account. A file containing the JSON key will download to your computer.
          1.  __Note:__ the JSON key is a private credential. Anyone holding the key can act as your Spoke service account, so keep it secure and *do not* check it into a source code repository! If the key gets into the wrong hands, you can disable it on your service account and generate a new key.
      1.  Make note of the `client_email` and `private_key` fields from the JSON key. An easy way to do this is with [jq](https://stedolan.github.io/jq/):
          ```shell
          $ jq .client_email <path to key file>
          $ jq .private_key <path to key file>
          ```
      1.  Configure environment variables:
      | Name | Value |
      | ---- | ----- |
      | `GCP_SERVICE_ACCOUNT_EMAIL` | value of the `client_email` field of the JSON key |
      | `GCP_SERVICE_ACCOUNT_PRIVATE_KEY` | value of the `private_key` field of the JSON key |
      | `GCP_STORAGE_BUCKET_NAME` | name of the bucket you created |
      | `GCLOUD_PROJECT` | if the Spoke service account and GCS bucket are in different projects, set this to the project ID which contains the GCS bucket. Otherwise leave unset |
          1.  Note that the private key will contain newline `\n` characters, so make sure you set your environment in a way that will handle those characters properly and not try to add any escaping.
