# Scheduled Backups

This example shows how to use Cloud Scheduler and Cloud Functions to configure a
schedule that creates Cloud Bigtable backups periodically.

The idea is to have a Cloud Scheduler job that invokes a Cloud function by
sending a message to the Pub/Sub topic which contains information about the
Cloud Bigtable backup creation request. Then the Cloud function initiates a
backup using Cloud Bigtable Java API.

## Create scheduled backups

1.  Clone this directory and make changes to
    `./config/scheduled-backups.properties` file to match your configuration.
    Fields that need to be updated start with `"replace"`.

2.  Create a Cloud Pub/Sub topic `cloud-bigtable-scheduled-backups` that serves
    as the target of the Cloud Scheduler job and triggers the Cloud function.
    For example:

```
gcloud pubsub topics create cloud-bigtable-scheduled-backups --project <project-id>
```

1.  Create and deploy a Cloud Function `cbt-create-backup-function` which is
    called whenever a Pub/Sub message arrives in
    `cloud-bigtable-scheduled-backups` topic:

```
./scripts/scheduled_backups.sh deploy-backup-function
```

1.  Deploy the scheduled backup configuration to Cloud Scheduler:

```
./scripts/scheduled_backups.sh create-schedule
```

## Email notification of backup failures

To get email notifications on backup creation failures, follow these steps:

1.  Follow this
    [guide](https://cloud.google.com/monitoring/support/notification-options#email)
    to add your email address as a notification channel.

2.  Create and deploy a custom metrics configuration file to filter logs
    generated by Cloud Functions, Cloud Scheduler and Cloud Bigtable. We use
    [Deployment Manager](https://cloud.google.com/deployment-manager/docs/quickstart)
    to create custom metrics. The example file can be found in
    `./config/metrics.yaml`. Deploy the custom metrics in Cloud Logging:

```
./scripts/scheduled_backups.sh add-metrics
```

After this, you should see two user-defined metrics under `Logs-based Metrics`
in Cloud Logging.

1.  Go to `Logs-based Metrics` in Cloud Logging and select `Create alert from
    metric` option for each of the two metrics created in the above step. From
    there, you can choose `Aggregrator`, such as `sum` or `mean`, for the target
    metric, and define what the condition of triggering an alert is, e.g., any
    time series violates that the value is above 0 for 1 minute.

Then add notification channels you just created to alerting policies.

### APIs and IAM roles setup

The diagram below shows how actions flow between human roles and APIs.
<img src="https://drive.google.com/uc?export=view&id=1YHUh5FKSuNMTSj6_E7Ehsq31RHNPu2Wu" width="600" height="auto" />

#### IAM Roles for Administrators

The administrator should be granted specific roles to deploy the services needed
for the solution.

|Role                                   |Purpose                                             |
|---------------------------------------|----------------------------------------------------|
|<em>roles/bigtable.admin</em>          |Cloud Bigtable Administrator                        |
|<em>roles/cloudfunctions.admin</em>    |to deploy and manage Cloud Functions                |
|<em>roles/deploymentmanager.editor</em>|to deploy monitoring metrics                        |
|<em>roles/pubsub.editor</em>           |to create and manage Pub/Sub topics                 |
|<em>roles/cloudscheduler.admin</em>    |to setup a schedule in Cloud Scheduler              |
|<em>roles/appengine.appAdmin</em>      |for Cloud Scheduler to deploy a cron service        |
|<em>roles/monitoring.admin</em>        |to setup alerting policies for failure notifications|
|<em>roles/logging.admin</em>           |to add log based user metrics to track failures     |


You also need a custom role (ie. backups-admin) with below permissions *
<em>appengine.applications.create</em> - for Cloud Scheduler to create an App
Engine app * <em>serviceusage.services.use</em> - for Cloud Scheduler to use the
App Engine app

#### Service Account for Cloud Functions

Cloud Functions calls Cloud Bigtable API to create a backup, it gets triggered
when a message arrives on the Pub/Sub topic. For successful execution of the
cloud function, it should be able to consume from the Pub/Sub topic and should
have permissions to create Cloud Bigtable backups. To accomplish this, perform
the following steps:

1.  Create a Service Account (e.g.
    cbt-scheduled-backups@<PROJECT>iam.gserviceaccount.com).
2.  Create a custom role (e.g. backups-admin) with the permissions:
    *   bigtable.backups.create
    *   bigtable.backups.delete
    *   bigtable.backups.get
    *   bigtable.backups.list
    *   bigtable.backups.restore
    *   bigtable.backups.update
    *   bigtable.instances.get
    *   bigtable.tables.create
    *   bigtable.tables.readRows
3.  Assign the custom role and <em>roles/pubsub.subscriber</em> to the service
    account. This allows Cloud Functions to read messages from the Pub/Sub topic
    and initiate a create backup request.
4.  Add the administrator as a service account user by adding the user as a
    member of the service account with role
    <em>roles/iam.serviceAccountUser</em>. This allows the administrator to
    deploy Cloud Functions.

### Limitations

To use Cloud Scheduler, you must
[create an App Engine app](https://cloud.google.com/scheduler/docs#supported_regions).
Once you set a zone for the App Engine app, you cannot change it. Your Cloud
Scheduler job will be running in the same zone as your App Engine app.
