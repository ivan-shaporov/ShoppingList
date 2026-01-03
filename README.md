# Shared Shopping List

> **Disclaimer**: This project is 100% vibecoded.

A simple, persistent, real-time shopping list web app that works on Android and iPhone without installation.

## Features

* **Shared State**: Syncs between multiple users in near real-time.
* **Zero Backend Code**: Connects directly from the browser to Azure Table Storage using a SAS token.
* **Privacy Focused**: Your data lives in your own Azure Storage account.

## Architecture

* **Frontend**: Single HTML file (`index.html`) with vanilla JavaScript.
* **Backend**: Azure Table Storage (No API servers, Functions, or App Services required).
* **Security**: Shared Access Signature (SAS) token passed via URL hash (client-side only).

## Setup

### 1. Azure Setup

1. Create an **Azure Storage Account**.
2. Create a **Table** (e.g., named `shopping-list`).
3. Generate a **SAS Token** for the table with the following permissions:
    * **Allowed Services**: Table
    * **Allowed Resource Types**: Service, Container, Object
    * **Allowed Permissions**: Read, Add, Update, Delete (Write is not strictly needed if you use Update/Merge, but good to have).
    * **Expiry**: Set it for a long duration (e.g., 1 year).
4. Copy the **Table Service SAS URL**. It should look like:
    `https://<your-account>.table.core.windows.net/shopping-list?sv=...&sig=...`
5. **Enable CORS**: By default, browsers block requests to Azure Table Storage. You must enable CORS on your Storage Account.
    * Go to **CORS** settings in your Storage Account (under Settings > Resource sharing or CORS).
    * Add a rule for the **Table service**:
        * **Allowed origins**: `*` (or your specific domain like `https://myapp.pages.dev`).
        * **Allowed methods**: `GET`, `PUT`, `MERGE`, `OPTIONS`.
        * **Allowed headers**: `*`.
        * **Exposed headers**: `*`.
        * **Max age**: `86400`.

### 2. Deployment

1. Upload `index.html` to any static hosting provider (GitHub Pages, Azure Static Web Apps, or even an Azure Storage Blob container).
2. Construct your access URL by appending your SAS URL to the **URL Hash** (using `#`), NOT the query string (using `?`).

   *   **CORRECT (Hash)**: `.../index.html#https://...` -> Secure. Token stays in browser.
   *   **INCORRECT (Query)**: `.../index.html?https://...` -> Insecure. Token sent to server.

   ```text
   https://<your-website>/index.html#<YOUR_SAS_URL>
   ```

   **Example:**
   `https://myshoppinglist.web.core.windows.net/index.html#https://mystorage.table.core.windows.net/shoppinglist?sv=2019-02-02&sig=...`

3. Bookmark this URL on your devices. The SAS token remains in your browser's hash fragment and is never sent to the hosting server.

## Managing Items

* **Editing**: The app currently does not support adding, editing, or deleting items via the UI.
* **Admin**: You can manage the list items (add new ones, change text, delete old ones) directly in the Azure Table using tools like [Microsoft Azure Storage Explorer](https://azure.microsoft.com/en-us/products/storage/storage-explorer/) or the Azure Portal.
