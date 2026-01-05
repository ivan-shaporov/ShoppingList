# Synclist

> **Disclaimer**: This project is 100% vibe-coded.

A simple, persistent, real-time checklist web app that works in a browser.

Perfect for **Shared Shopping Lists**.

* **Scenario**: You add "Milk" to the list at home. Your partner sees it appear instantly on their phone at the store and checks it off.

## Features

* **Shared State**: Syncs between multiple users in near real-time.
* **Zero Backend Code**: Connects directly from the browser to Azure Table Storage using a SAS token.
* **Privacy Focused**: Your data lives in your own Azure Storage account.
* **Edit Mode**: Toggle between a clean view-only list and an editable interface.
* **Multiple Lists**: Items are grouped by list name (Azure Table `PartitionKey`).

## Architecture

* **Frontend**: Single HTML file (`index.html`) with vanilla JavaScript.
* **Backend**: Azure Table Storage (No API servers, Functions, or App Services required).
* **Security**: Shared Access Signature (SAS) token passed via URL hash (client-side only).

## Setup

### 1. Automated Deployment

This project includes a Bicep template that automates the entire infrastructure setup:

* **Creates an Azure Storage Account**.
* **Configures CORS for Azure Table Storage**:
  * Allowed Origins: `*` (Allows access from any domain, including your static site).
  * Allowed Methods: `GET`, `HEAD`, `MERGE`, `POST`, `OPTIONS`, `PUT`, `DELETE`.
  * Max Age: `200` seconds.
* **Creates the `synclist` table**.
* **Sets up Stored Access Policies** (Best Practice):
  * `webfulledit`: Permissions `raud` (Read, Add, Update, Delete). Use this for Edit mode.
  * `webqueryupdate`: Permissions `ru` (Read, Update). Use this for standard View mode.
  * **Duration**: Defaults to 1 year from deployment. Configurable via parameters.
  * *Benefit*: Generating SAS tokens linked to a policy allows you to revoke access later by modifying the policy in Azure, without changing the URLs on everyone's devices.
* **Enables Static Website hosting**.
* **Uploads `index.html`** to the `$web` container.

**Prerequisites:**

* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) installed and logged in (`az login`).

**Deploy:**

### Option 1: VS Code Task

1. Open the Command Palette (`Ctrl+Shift+P` or `Cmd+Shift+P`).
2. Type **"Tasks: Run Task"**.
3. Select **"Deploy Bicep"**.
4. Enter the **Resource Group** name (e.g., `synclist`).
5. Enter a globally unique **Storage Account Name** (e.g., `mysharedlist123`).

### Option 2: Azure CLI

1. Open a terminal in the project folder.
2. Run the deployment command:

   ```bash
   # Default deployment (Policies expire in 1 year)
   az deployment group create \
     --resource-group <your-resource-group> \
     --template-file main.bicep \
     --name deploy \
     --parameters storageAccountName=<unique-storage-name>
   ```

3. The command will output the **Static Website Endpoint** (e.g., `https://<storage-account>.z22.web.core.windows.net/`).

### 2. Generate SAS Token

After deployment, you need to generate a SAS token to allow the app to access the database. The Bicep template has already created the necessary security policies (`webfulledit` and `webqueryupdate`).

1. Go to the **Azure Portal** > Your Storage Account > **Storage Browser** > **Tables**.
2. Select the table (e.g., `synclist`).
3. Click **Access Policy** to verify the policies exist.
4. Use **Azure Storage Explorer** or the **Azure Portal** to generate a SAS token:

   * **Signing method**: SAS key / Account Key.
   * **Table**: Select your table.
   * **Access Policy**: Select `webfulledit` (for full access) or `webqueryupdate` (for standard access).
   * **Generate**.

5. Copy the **Table Service SAS URL** (the full URL, e.g., `https://<account>.table.core.windows.net/synclist?sv=...`).

### 3. Construct Access URL

Combine your Static Website URL with the SAS Token using a **Hash** (`#`), **NOT** a Query String (`?`).

* **CORRECT (Hash)**: `.../index.html#https://...` -> **Secure**. Token stays in the browser and is NOT sent to the server.
* **INCORRECT (Query)**: `.../index.html?https://...` -> **Insecure**. Token is sent to the server in the HTTP request.

Format:
`[Static Website URL]/#[SAS Token]`

Example:
`https://mychecklist.z22.web.core.windows.net/#https://mystorage.table.core.windows.net/synclist?sv=2019-02-02&tn=synclist&sig=...`

> **Tip**: Bookmark this URL on your devices. The SAS token remains in your browser's hash fragment and is never sent to the hosting server.

## Usage

## Lists (Partitions)

This app supports multiple independent lists inside the same Azure Table:

* Each list is an Azure Table **`PartitionKey`**.
* The UI renders **one section per list**.

In edit mode, each list section has its own “Add new item…” input. There is also a global adder (“List name…” + “Add new item…”) that is only shown when the SAS URL is not restricted to a single list.

### View Mode

By default, the list is in "View Mode". You can check and uncheck items, but you cannot add or delete them. This is great for execution phase to avoid accidental edits.

URL format:
`https://<your-website>/index.html#<YOUR_webqueryupdate_SAS_URL>`

### Edit Mode

To add or delete items, append `?edit` **before** the hash (`#`).

URL format:
`https://<your-website>/index.html?edit#<YOUR_webfulledit_SAS_URL>`

* **Add Item**: Type in the text box and press Enter or click "Add".
* **Delete Item**: Click the "Delete" button next to an item (requires confirmation).

  **Example:**
  `https://mychecklist.web.core.windows.net/index.html#https://mystorage.table.core.windows.net/synclist?sv=2019-02-02&sig=...`

## Managing Items

* **Editing**: The app currently does not support editing items via the UI.
* **Admin**: You can manage the list items (add new ones, change text, delete old ones) directly in the Azure Table using tools like [Microsoft Azure Storage Explorer](https://azure.microsoft.com/en-us/products/storage/storage-explorer/) or the Azure Portal.

## Development

### Updating the App

If you make changes to `index.html`, you can upload the new version without re-deploying the entire infrastructure.

**Using VS Code Task:**

1. Open the Command Palette (`Ctrl+Shift+P` or `Cmd+Shift+P`).
2. Type **"Tasks: Run Task"**.
3. Select **"Upload index.html"**.
4. Enter the **Storage Account Name** when prompted.

## Sharing One List vs All Lists

You can deploy once and share either:

* **All lists** (multi-list access): generate a normal Table SAS URL for the table.
* **A single list** (one `PartitionKey` only): generate a SAS URL that is partition-scoped.

Partition-scoped Table SAS URLs commonly include parameters like `spk`/`epk` or `sprk`/`eprk` in the SAS query string (tooling varies). When these are present, the app hides the global adder because the URL is intentionally restricted to one list.

How to generate a single-list link (high level):

1. In Azure Storage Explorer (or your SAS generator), choose your table and create a SAS.
2. Set the **start/end partition key** (or equivalent) to the list name you want to share.
  Caveat: when you scope a Table SAS to a **partition key range**, Azure also requires a **row key range**.
  When you set start/end partition key, also set start/end row key.
  Use `0` for start row key and `A` for end row key. Using characters like `!` and `~` caused update operations to fail in our testing.
3. Use that SAS URL in the normal hash format.
