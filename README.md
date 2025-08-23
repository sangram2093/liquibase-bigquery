### **Guide: Connecting DBeaver to BigQuery via Workload Identity Federation (WIF)**

**Objective:**
This document outlines the standard procedure for securely connecting DBeaver to Google BigQuery using Workload Identity Federation (WIF). This modern authentication method eliminates the need for long-lived service account JSON keys, enhancing security by leveraging short-lived, federated credentials.

-----

### **Prerequisites**

Before you begin, ensure you have the following:

* **DBeaver Version:** `25.1.5` or newer. Older versions lack the necessary support for WIF authentication and will fail.
* **Google Cloud SDK:** The `gcloud` command-line tool must be installed and authenticated on your local machine.
* **Required Information:** You must have the following details for your target GCP environment:
    * GCP Project ID
    * Workload Identity Pool ID
    * Workload Identity Provider ID
    * The email of the Service Account you will impersonate (e.g., `my-sa@my-project.iam.gserviceaccount.com`)
* **JWT Token Access:** You must have access to the internal automation for generating a valid JWT identity token.

-----

### **Configuration Steps**

Follow these steps to configure your DBeaver client.

#### **Step 1: Obtain Your JWT Token**

The WIF process requires a temporary identity token from a trusted external source.

1.  Access the internal endpoint or run the required automation to generate a fresh JWT token.
2.  Save the complete token content to a plain text file on your local machine. For this guide, we will refer to this file as `jwt_token.txt`.

> **Note:** These JWT tokens are short-lived. If your connection fails with an authentication error, your first troubleshooting step should be to regenerate this token.

#### **Step 2: Generate the WIF Credential Configuration File**

Next, use `gcloud` to create a special configuration file that tells Google's drivers how to use your local JWT to authenticate.

1.  Open your terminal or command prompt.

2.  Run the following command, replacing the placeholders with your specific information:

    ```bash
    gcloud iam workload-identity-pools create-cred-config \
      "locations/global/workloadIdentityPools/<POOL_ID>/providers/<PROVIDER_ID>" \
      --service-account="<SERVICE_ACCOUNT_EMAIL>" \
      --output-file="<PATH_TO_SAVE>/external_account.json" \
      --credential-source-file="<PATH_TO>/jwt_token.txt"
    ```

    * `<POOL_ID>`: Your Workload Identity Pool ID.
    * `<PROVIDER_ID>`: Your Workload Identity Provider ID.
    * `<SERVICE_ACCOUNT_EMAIL>`: The full email of the service account to impersonate.
    * `<PATH_TO_SAVE>/external_account.json`: The full path where the output file will be saved (e.g., `C:\gcp\creds\external_account.json`).
    * `<PATH_TO>/jwt_token.txt`: The full path to the token file you created in Step 1.

    This command creates the `external_account.json` file, which acts as a "pointer" rather than a traditional key file.

#### **Step 3: Configure the DBeaver BigQuery Driver**

This one-time setup modifies DBeaver's global BigQuery driver to support the WIF parameters.

1.  Open DBeaver and navigate to **Database \> Driver Manager**.

2.  Select **Google BigQuery** from the list and click **Edit**.

3.  Go to the **Settings** tab.

4.  Locate the **URL Template** field and replace its content with the following string. This new template adds the required `BYOID` (Bring Your Own ID) parameters for WIF.

    ```
    jdbc:bigquery://https://www.googleapis.com/bigquery/v2:443;ProjectId={project};OAuthType=4;DefaultDataset={database};Location={server};timeout=3600;BYOID_AudienceUri=//iam.googleapis.com/locations/global/workloadIdentityPools/<POOL_ID>/providers/<PROVIDER_ID>;BYOID_TokenUri=https://sts.googleapis.com/v1/token;BYOID_SA_Impersonation_Uri=https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/<SERVICE_ACCOUNT_EMAIL>:generateAccessToken;BYOID_CredentialSource={"file":"<PATH_TO>/jwt_token.txt"}
    ```

    **Important:** You must replace the placeholders in the template with your **actual values**. While the connection will still ask for these details, having them in the template ensures correctness.

5.  Click **OK** to save the driver settings.

#### **Step 4: Create the New BigQuery Connection**

1.  In the DBeaver main window, go to **Database \> New Database Connection**.

2.  Select **Google BigQuery** and click **Next**.

3.  On the **Main** tab, configure the following:

    * **Project:** Enter your GCP Project ID.
    * **Authentication:** Select **Service Account Key**.
    * **Key Path:** Click **Browse** and select the `external_account.json` file you generated in Step 2.

4.  Configure any necessary proxy settings in the **Proxy** tab as required by your organization's network policy.

#### **Step 5: Test and Connect**

1.  Click the **Test Connection** button at the bottom left.
2.  If all steps were performed correctly, you should see a "Success" dialog.
3.  Click **Finish** to save the connection.

You have now successfully configured DBeaver to securely access BigQuery without static credentials. Remember to regenerate your `jwt_token.txt` file when it expires.
