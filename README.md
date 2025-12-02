# Migrate VMs to Compute Engine and RDS databases to Cloud SQL

This document outlines the process of migrating virtual machines (VMs) and relational database service (RDS) instances from Amazon Web Services (AWS) to Google Cloud Platform (GCP). It details the steps for migrating an application running on AWS EC2 to Compute Engine, and an AWS RDS PostgreSQL database to Cloud SQL for PostgreSQL, utilizing Google Cloud's migration services.

## Requirements

To deploy this demo, you need:

-   An AWS environment with appropriate permissions for EC2, RDS, VPC, and IAM. Specifically, the AWS credentials should have:
-   A Google Cloud project with `Owner` or `Editor` roles.
-   [Terraform CLI >= 1.0](https://developer.hashicorp.com/terraform/install)
-   [Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

### Migrate VMs and Databases from AWS

In this demo, you migrate an application running on AWS to Google Cloud.

This process involves:

1.  Discover your AWS assets in the Migration Center.
1.  Generate Total Cost of Ownership (TCO) and asset reports.
1.  Migrate an RDS PostgreSQL to Cloud SQL using Database Migration Service
    (DMS).
1.  Migrate Amazon EC2 VMs to Compute Engine using Migrate to Virtual Machines
    (M2VM).

### **Deploy AWS Infrastructure**

To prepare the source environment, open your terminal and deploy the three-tier

application on AWS using Terraform.

1.  Set the working directory:

    ```bash
    cd "$(git rev-parse --show-toplevel)/projects/migrate-from-aws-to-google-cloud/vm-compute-db-modernization/terraform"
    ```

1.  Initialize and deploy the Terraform configuration:

    ```bash
    cp terraform.tfvars.example terraform.tfvars
    terraform init
    terraform apply --auto-approve
    ```

1.  Capture Output Details: Upon completion, Terraform displays a summary
    containing critical values. Keep this output visible or save it, as you will
    need the RDS Endpoint, Instance ID, and Security Group ID later.
1.  Verify Deployment: Wait 3-5 minutes for the application to initialize, then
    test the health endpoint:

    ```bash

    APP_URL=$(terraform output -raw app_url)
    curl $APP_URL/health

    ```

1.  Expected output: {"database":"connected","status":"healthy"}.

### Assess with Migration Center

Use Google Cloud Migration Center to discover assets and generate reports that

inform your migration strategy.

1.  Configure Environment Variables: In your terminal, export your credentials
    to be used by the setup script:

    ```bash
    export GCP_PROJECT_ID=[YOUR_GCP_PROJECT]
    export AWS_SECRET_ACCESS_KEY=[YOUR_AWS_SECRET_KEY]
    export AWS_ACCESS_KEY_ID=[YOUR_AWS_ACCESS_KEY_ID]
    ```

1.  **Run Setup Script:** Execute the helper script to prepare your project:

    ```sh
    ../scripts/migration_center_aws_setup.sh
    ```

- Enables APIs: Activates required services (Migration Center, Compute, Cloud
  SQL, etc.).
- Creates Service Account: Provisions aws-discovery-sa to run the discovery job.
- Secures Credentials: Safely stores your AWS_SECRET_ACCESS_KEY in Secret
  Manager (aws-discovery-secret).
- Grants Permissions: Binds the necessary IAM roles (migrationcenter.admin,
  secretmanager.secretAccessor) to the service account.

1.  Initialize Migration Center: In the Google Cloud Console, navigate to
    Migration Center and click Get Started.

- **Region:** Select **us-east4** and click **Next**.
- Account: Click Add custom account. In Account Name, enter TEST.
- Opportunity: Click Add custom opportunity. In Opportunity Name, enter TEST.
- Click **Next**.
- Scroll to the bottom of the page, click **Next**, then **Continue**.

#### Run Discovery

1 Navigate to **Data import** \> **Add data** \> **Direct AWS discovery**. -
Center AWS Discovery SA service account (created by the script).

- Click **Start AWS discovery**

1.  Monitor Progress

1.  To view the detailed status of the import job, navigate to Cloud Run \>
    Jobs.
    - Wait for the job to complete successfully.
    - Once the Cloud Run job is successful, navigate back to Migration Center \>
      Assets to view the discovered AWS resources.
1.  Generate and Analyze Reports: Once discovery is complete, generate reports
    to guide your planning:
    - TCO Report: Go to Reports \> Create reports \> TCO and detailed pricing
      reports.
    - Asset Inventory: Go to Assets \> Export. Download the asset details (CSVs)
      to inspect technical specifications like vCPU count and RAM to ensure you
      select the correct targets in Google Cloud.

### Plan the Migration

Using the data gathered in the assessment phase, you create a mapping plan.

This ensures your target infrastructure is "right-sized" and compatible.

1.  Analyze Database Requirements: The asset report identifies the source as
    PostgreSQL 16.9 with 2 vCPUs. To ensure compatibility and performance, map
    this to Cloud SQL for PostgreSQL 16\.
1.  Analyze Compute Requirements: The asset report identifies the source VM
    (mmb-app-app) as a t3.small (2 vCPU, 2 GB RAM) running Ubuntu. Map this to a
    Google Cloud e2-small instance, which provides equivalent capacity for
    general-purpose workloads.

### Migrate

#### Migrate Database (AWS RDS to Cloud SQL)

Use the Database Migration Service (DMS) to replicate data from AWS to Google

Cloud.

1.  Download AWS RDS Certificate: To enable secure TLS authentication, download
    the certificate bundle for the us-east-1 region:

    ```bash
    wget [https://truststore.pki.rds.amazonaws.com/us-east-1/us-east-1-bundle.pem](https://truststore.pki.rds.amazonaws.com/us-east-1/us-east-1-bundle.pem)
    ```

1.  This will download the certificates as us-east-1-bundle.pem.
1.  Create Connection Profile (Source): In the Google Cloud Console, navigate to
    Database Migration Service \> Connection profiles.
    - Click **Create profile**.
    - Database engine: Select Amazon RDS for PostgreSQL.
    - Connection profile name: Enter aws-postgres-source.
    - Region:\*Selectus-east1.
1.  Define connection configurations:
    - Select **PostgreSQL to PostgreSQL**.
    - Click **Define**.
    - Hostname or IP address: Enter the RDS Endpoint. Retrieve this by running
      terraform output \-raw rds_endpoint in your terminal.
    - Port: Enter 5432.
    - Username/Password: Enter dms_user and SecureDmsPass123\!.
    - Encryption type: Select TLS authentication\.
    - CA Certificate: Upload the us-east-1-bundle.pem file you downloaded
      earlier.
    - Click **Save**
    - Click **Create**

1.  Create Migration Job & Destination Instance:
    - Navigate to **Migration jobs** and click **Create Migration Job**.
    - Migration job name: Enter mmb-db-migration.
    - **Source database engine:** Select **Amazon RDS for PostgreSQL**.
    - Destination database engine: Select Cloud SQL for PostgreSQL.
    - **Destination region:** Select **us-east1**.
    - Click **Save & Continue**.

1.  Define Source
    - Select the **aws-postgres-source** connection profile.
    - Click **Save & Continue**.

1.  Create Destination
    - Select **New Instance**
    - **Instance ID:** Enter postgres-instance.
    - **Password:** Enter SecureAppPass123\!.
    - **Database version:** Select **Cloud SQL for PostgreSQL 16**.
    - **Choose a Cloud SQL edition:** Select **Enterprise**.
    - **Configure your instance:**
    - Select **Private**.
    - Click **Allocation IP range** and **Private Id**.
    - Click **Create & Continue**.
    - Wait: Wait for the Database instance creation to finish before moving on
1.  Define connectivity method:
    - elect **IP allowlist**.
    - Copy the IP addresses displayed (you will need these for the next step).

#### Update Security Group (AWS)

1.  Go to the **AWS Console** and navigate to the **Aurora and RDS** page.
    - Click **Databases** and select the database **my-app-db**.
    - Under Connectivity & security, click on the Security group link.
    - Click **Edit inbound rules** \> **Add rule**.
    - **Type:** Select **PostgreSQL**.
    - Source: Paste the IP addresses you copied from the Google Cloud Console.
    - Click **Save rules**.
1.  Configure Parameter Group (AWS): While still in the AWS Console (Aurora and
    RDS):
    - Select **Parameter groups** from the left navigation.
    - Select the parameter group used by your instance.
    - In the search box, enter shared_preload_libraries.
    - Select the parameter and click **Edit parameters**.
    - Add ,pglogical to the end of the existing values (e.g.,
      rds_utils,pglogical).
    - Click **Save changes**.

1.  Enable pglogical Extension (AWS EC2): Use the application server to enable
    the extension and grant required permissions to the migration user.
    - In the AWS Console, go to **EC2** \> **Instances**.
    - Select your app instance (mmb-app-app or similar).
    - Click **Connect** (top right).
    - Select the **Session Manager** tab and click **Connect**.
    - In the terminal window that opens, run the following commands:

    ```bash
    RDS_ENDPOINT=$(cat /opt/flask-app/.env | grep DB_HOST | cut -d= -f2)
    export PGPASSWORD='SecureAppPass123!'

    psql -h $RDS_ENDPOINT -U postgres -d appdb <<EOF
    CREATE EXTENSION pglogical;
    GRANT USAGE ON SCHEMA pglogical TO dms_user;
    GRANT ALL ON ALL TABLES IN SCHEMA pglogical TO dms_user;
    GRANT ALL ON ALL SEQUENCES IN SCHEMA pglogical TO dms_user;
    SELECT nspname, pg_catalog.has_schema_privilege('dms_user', nspname, 'USAGE') as has_usage
    FROM pg_namespace WHERE nspname = 'pglogical';
    EOF
    ```

1.  **Reboot Database (AWS):** Return to the **Aurora and RDS** console.
    - Navigate to **Databases**, select **my-app-db**.
    - Click **Actions** \> **Reboot**.
    - Wait for the status to return to **Available**.
1.  **Start the Migration Job (Google Cloud):**
    - Navigate back to the Google Cloud Console.
    - In the **Database to migrate** dropdown, select **Specific database**.
    - Select the database named **appdb**.
    - Click **Create & Start Job**.

1.  **Promote the Migration:**
    - Monitor the job status.
    - When the status changes to CDC (Change Data Capture) and the lag is near
      zero, click Promote.
    - Confirm the promotion. This disconnects the AWS source and makes the Cloud
      SQL instance is your primary, standalone database.

#### Migrate Virtual Machine (EC2 to Compute Engine)

Use Migrate to Virtual Machines (M2VM) to replicate the application server.

1.  Create AWS Source: In the Google Cloud Console, you must create a source to
    connect M2VM to your AWS account.
1.  Navigate to **Migrate to Virtual Machines**, select the **SOURCES** tab.
1.  Click **ADD SOURCE** and select **\+Add AWS Source**.
1.  Enter the source details:
    - **Name:** aws-source
    - GCP region: us-east1 (This is where your replication infrastructure will
      be created).
    - AWS region: us-east-1 (This is where your EC2 instance is located).
    - **AWS Access key ID:** Enter your access key.
    - **AWS Secret access key:** Enter the aws access secret. Click **Create**.
      Wait: It may take up to 15 minutes for the Source status to become Active.
    - Note: If you attempt to view inventory early, you may see a message
      stating "Source VM inventory is not available while the source is in the
      Pending status." This is normal behavior; please wait for the status to
      change to Active.

1.  Configure Target Project: Before starting replication, explicitly configure
    the destination project.
    - In the Migrate to Virtual Machines dashboard, select the TARGETS tab.
    - If your current project is not already listed or selected, click ADD
      TARGET PROJECT.
    - Select the current project ID from the list to ensure the VMs are migrated
      to this project.

1.  Start Replication: Start the replication process to copy the VM data to
    Google Cloud.

1.  Navigate to the **MIGRATIONS** > **Add migration** tab.
    - In the list of instances, filter by the Instance ID (retrieve this via
      terraform output \-raw app_instance_id).
    - Select the mmb-app-app instance and click **Add migrations**.
    - Select the newly added migration and click **Edit target details**

1.  Configure the target settings:
    - **Machine type:** e2-small (as determined in the Plan phase).
    - **Project:** Your current project.
    - **Zone:** us-east1-b (or your preferred zone).

1.  Click **Save**.
1.  Select the migration again and click **Start replication**.
1.  **Cutover:** Finalize the migration to create the Compute Engine instance.
    - Wait for the replication status to reach **Active (Idle)**.
    - Click **Cutover**.
    - In the confirmation panel, review the details and click Start Cutover.
    - Wait for the cutover job to complete. The VM is now running in Google
      Cloud.

### **Configure Application and Load Balancing**

Update the migrated VM to connect to the new database and expose it via a Load
Balancer.

1.  **Update Connectivity:** In your terminal, SSH into the migrated VM:

    ```bash
    gcloud compute ssh mmb-app-app --zone=us-east1-b
    ```

1.  Update /opt/flask-app/.env to set DB_HOST to the Private IP of the Cloud SQL
    instance. Restart the app: sudo systemctl restart flask-app.
1.  Deploy Load Balancer: Run the following to configure the global HTTP Load
    Balancer:

    ```bash
    ./deploy_loadbalancer.sh
    ```

1.  **Verify:** Get the LB IP: gcloud compute forwarding-rules describe
    flask-http-rule \--global \--format='value(IPAddress)' and test with:

    ```bash
    curl http://[LB_IP]/health
    ```

### Cleanup

To clean up the resources created in this chapter:

1.  M2VM: Finalize the migration in the Google Cloud Console.
1.  AWS: Run terraform destroy \-auto-approve in the aws-terraform directory.
1.  GCP: Delete the Forwarding Rule, VM, and Cloud SQL instance using gcloud
    commands or the Console.
