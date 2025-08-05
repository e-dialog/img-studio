# Infrastructure setup guide for ImgStudio (Public Version)

This guide details how to deploy a **publicly accessible** version of ImgStudio that does not require user authentication.

## 0\. Setup **Vertex** & Request access to **required models**

- **General Vertex AI Setup**
  - Go to `Vertex AI` > `Enable all recommended APIs`. This typically includes:
    - `Vertex AI API`
    - `Cloud Storage API`
  - Confirm that the Vertex Service Account exists in your project. It should look like this:`service-PROJECT_NUMBER@gcp-sa-aiplatform.iam.gserviceaccount.com`

- **Imagen Models**
  - **Children (minors) generation**: contact your commercial team to ask for access
  - **Imagen 4** Generation
    - **Public Preview:** `imagen-4.0-generate-preview-06-06` (Standard)
    - **Public Preview:** `imagen-4.0-fast-generate-preview-06-06` (Fast)
    - **Public Preview:** `imagen-4.0-ultra-generate-preview-06-06` (Ultra)
  - **Imagen 3** Editing & Customization
    - **Private Access:** `imagen-3.0-capability-001` - to gain access, fill out [this form](https://docs.google.com/forms/d/e/1FAIpQLScN9KOtbuwnEh6pV7xjxib5up5kG_uPqnBtJ8GcubZ6M3i5Cw/viewform)
  - **Image Segmentation model** (required for image editing in app)
    - **Private Access:** `image-segmentation-001` - to gain access, fill out [this form](https://docs.google.com/forms/d/e/1FAIpQLSdzIR1EeQGFcMsqd9nPip5e9ovDKSjfWRd58QVjo1zLpfdvEg/viewform?resourcekey=0-Pvqc66u-0Z1QmuzHq4wLKg&pli=1)

- **Veo Models**
  - **Veo 2** Generation (Text to Video, Image to Video)
    - **Public GA:** `veo-2.0-generate-001` (Standard)
  - **Veo 3** Generation (Text to Video+Audio)
    - **Public Preview:** `veo-3.0-generate-preview` (Standard)
    - **Public Preview:** `veo-3.0-fast-generate-preview` (Fast)
  - **Veo 3 Standard** ITV Generation (Image to Video + Audio) & **Veo 2** Advanced features (Interpolation, Camera Preset Controls, Video Extend)
    - **Private Access:** `veo-3.0-generate-preview` & `veo-2.0-generate-exp` - to gain access, fill out [this form](https://docs.google.com/forms/d/e/1FAIpQLSciY6O_qGg2J0A8VUcK4egJ3_Tysh-wGTl-l218XtC0e7lM_w/viewform)

## 1\. Create **Cloud Storage** buckets

- **Specifications:** Regional in your desired region (ex: `europe-west9` in Paris)
- **Create 3 buckets**
  - **Raw generated output content**: `YOUR_COMPANY-imgstudio-output`
  - **Shared content**: `YOUR_COMPANY-imgstudio-library`
  - **Configuration file bucket**: `YOUR_COMPANY-imgstudio-export-config`
    - Here upload `export-fields-options.json` a **configuration file specific** to your usage that you can find an [example](https://github.com/aduboue/img-studio/blob/main/export-fields-options.json) of in the [repository](https://github.com/aduboue/img-studio), its purpose is to setup the desired **metadata** you want to set for your generated content
    - In this file, you can change the **ID**, **label**, **name**, **isMandatory** tag, and **options** for each field
    - Attention! The ID and the options’ values must only be letters, no spaces, no special characters, starting with a lowercase letter

## 2\. Setup **Cloud Build** trigger

- Name: `YOUR_COMPANY-imgstudio`
- **Event:** `Manual invocation`
- **Source:**
  - Go to the public repository [**https://github.com/aduboue/img-studio**](https://github.com/aduboue/img-studio)
  - **Monitor new releases** to stay up-to-date: click on `Watch` > `Custom` > `Releases` > Apply
  - Use the top right button to setup a **GitHub fork** from the repository, which will **create a copy repository in your own GitHub account**
  - Back in Cloud Build, **log into your GitHub account**, then select the newly created repository
- **Configuration:** `Cloud Build configuration file (yaml or json)`
  - Cloud Build configuration file location: `/cloudbuild.yaml`
- Put in the required **substitution variables:**
  - `_NEXT_PUBLIC_EXPORT_FIELDS_OPTIONS_URI`
    - The URI of the configuration JSON file in its bucket
    - Ex: `gs://YOUR_COMPANY-imgstudio-export-config/export-fields-options.json`
  - `_NEXT_PUBLIC_GCS_BUCKET_LOCATION`
    - The region selected for your GCS buckets
    - Ex: `europe-west9`
  - `_NEXT_PUBLIC_VERTEX_API_LOCATION`
    - The region you want to use for VertexAI APIs
    - Ex: `europe-west9`
  - `_NEXT_PUBLIC_GEMINI_MODEL` = `gemini-2.0-flash-001`
    - Don’t change this
  - `_NEXT_PUBLIC_OUTPUT_BUCKET`
    - The name of the raw generated output content bucket
    - Ex: `YOUR_COMPANY-imgstudio-output`
  - `_NEXT_PUBLIC_TEAM_BUCKET`
    - The name of the shared content bucket
    - Ex: `YOUR_COMPANY-imgstudio-library`
  - `_NEXT_PUBLIC_EDIT_ENABLED`
    - Allow to enable edit features, **set it to `false` if you do not have access yet**
  - `_NEXT_PUBLIC_SEG_MODEL`
    - **Only mandatory if Edit is enabled**
    - Service name for the Vertex Segmentation model, when you get access to it **(see Step 0)**
  - `_NEXT_PUBLIC_VEO_ENABLED`
    - Allow to enable Veo text-to-video generation
  - `_NEXT_PUBLIC_VEO_ITV_ENABLED`
    - Allow to enable Veo image-to-video generation, **set it to `false` if you do not have access yet (see Step 0)**
  - `_NEXT_PUBLIC_VEO_ADVANCED_ENABLED`
    - Allow to enable Veo advanced features (interpolation, camera preset), **set it to `false` if you do not have access yet (see Step 0)**
- **Service account:** select the **default already existing Cloud Build service account**
  - You may want to **check in IAM it has the roles**: `Artifact Registry Writer` and `Logs Writer`
- > Save
- **Manually run your first build !**

## 3\. Create application **Service Account**

- Go to IAM > Service Accounts > Create Service Account
- Provide name: `YOUR_COMPANY-imgstudio-sa`
- Give **roles**:
  - `Cloud Datastore User`
  - `Logs Writer`
  - `Secret Manager Secret Accessor`
  - `Service Account Token Creator`
  - `Storage Object User`
  - `Vertex AI User`

## 4\. Deploy **Cloud Run** service

- Deploy container > Service
- `Deploy one revision from an existing container image`
- **Container image** > Select from **Artifact registry** the `latest` image you just build in Cloud Build
- Name: `YOUR_COMPANY-imgstudio-app`
- Set your region (ex: `europe-west9`)
- **Authentication**: **Select "Allow unauthenticated invocations"**. This makes the service publicly accessible.
- **Ingress Control** > **Select "Allow all traffic"**. This allows users to access the service directly via its default URL.
- Container(s) **> Container port:** `3000`
- Security **> Service account:** `YOUR_COMPANY-imgstudio-sa`
- > Create

## 5\. **Firestore** Database creation

- Firestore > Create database
  - Firestore **mode**: `Native mode`
  - Database **ID**: `(default)` (**very important you keep it that way**)
  - Location type: `Region`
  - Region: your desired region (ex: `europe-west9` in Paris)
- Firestore > Indexes > **Composite indexes** > Create Index
  - **Collection ID**: `metadata`
  - **Fields to index**
    - Field path 1: `combinedFilters`, Index options 1: `Array contains`
    - Field path 2: `timestamp`, Index options 2: `Descending`
    - Field path 3: `__name__`, Index options 3: `Descending`
  - Query **scope**: `Collection`
  - > Create
  - **Wait for the index to be successfully created!**
- Let’s **setup security rules on your database** to allow the public application to read and write data.
  - In a new tab, go to `https://console.firebase.google.com/project/PROJECT_ID/firestore/databases/-default-/rules`
  - In the **security rules editor**, write the following content to allow public access:
    ```
    rules_version = '2';
    service cloud.firestore {
      match /databases/{database}/documents {
        match /{document=**} {
          allow read, write: if true;
        }
      }
    }
    ```
  - > Publish