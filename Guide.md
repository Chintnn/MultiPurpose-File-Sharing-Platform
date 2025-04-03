
# Building a Serverless File Sharing Platform on AWS

## 1. Introduction

This project extends an original serverless file sharing platform (which supported text files only) so that it now handles any file type (e.g. PDF, PNG, JPG, DOCX). You will learn how to:

- **Create an Amazon S3 bucket** for file storage.
- **Set up AWS Lambda functions (in Python)** to handle file uploads and downloads.
- **Expose these functions via Amazon API Gateway** with binary payload support.
- **Secure your architecture using IAM roles.**

The final solution lets you interact with the platform using HTTP clients such as cURL or Postman.

---

## 2. Prerequisites

Before you begin, make sure you have:
- An AWS account.
- AWS CLI installed and configured with your credentials.
- Basic familiarity with Python and terminal commands.
- (Optional) An IDE or text editor for editing Python code.

---

## 3. Project Architecture Overview

The project consists of the following components:

- **Amazon S3:** Stores files of any type.
- **AWS Lambda (Python):** Contains two functions:
  - **Upload Function:** Receives HTTP requests, decodes the file (binary or text), and uploads it to S3.
  - **Download Function:** Retrieves the file from S3, base64 encodes it, and returns it so that API Gateway can deliver it as binary data.
- **Amazon API Gateway:** Exposes RESTful endpoints for uploading and downloading files. Configured to support binary media types.
- **IAM:** Secures access policies for Lambda functions to interact with S3.

**Architecture Diagram:**

```
     [User / HTTP Client]
             |
         API Gateway  <------ (Configured with Binary Support)
         /           \
        /             \
[Upload Lambda]   [Download Lambda]
        \             /
         \           /
           Amazon S3 Bucket
```

---

## 4. Step-by-Step Setup

### Step 4.1: Create an S3 Bucket

1. **Sign in to AWS Management Console:**
   - Open your web browser and navigate to: [https://console.aws.amazon.com/](https://console.aws.amazon.com/).
   - Enter your credentials and log in.

2. **Navigate to S3:**
   - In the top search bar, type **“S3”** and click on the **“S3”** service from the drop-down list.

3. **Create a New Bucket:**
   - In the S3 dashboard, click the **“Create bucket”** button (usually in the upper-right corner).
   - On the “Create bucket” page:
     - In the **Bucket name** field, enter a unique name (e.g., `my-multi-type-file-share-bucket`). **(Tip: Bucket names must be globally unique and all lowercase.)**
     - In the **Region** drop-down, select your preferred region (for example, **“US East (N. Virginia)”**). 
     - Leave the other settings as default (you can modify these later if needed).
   - Scroll to the bottom and click **“Create bucket”**.

---

### Step 4.2: Set Up IAM Roles for Lambda

1. **Open the IAM Console:**
   - From the AWS Management Console, type **“IAM”** in the search bar and select **“IAM”**.

2. **Create a New Role:**
   - In the left navigation pane, click on **“Roles”**.
   - Click on the **“Create role”** button.
   - Select **“AWS service”** as the trusted entity and choose **“Lambda”** for the service that will use this role.
   - Click **“Next: Permissions”**.

3. **Attach Policy:**
   - Search for and select the **“AmazonS3FullAccess”** policy (or create a custom policy allowing `s3:PutObject` and `s3:GetObject` for your bucket).
   - Click **“Next: Tags”** (optional) and then **“Next: Review”**.

4. **Name and Create Role:**
   - Enter a role name such as **“LambdaS3AccessRole”**.
   - Click **“Create role”**.

5. **Attach Role to Lambdas Later:**  
   - This role will be attached when you create your Lambda functions.

---

### Step 4.3: Write the Lambda Functions

You will create two Lambda functions: one for uploading files and one for downloading files.

#### A. Upload Function (UploadFunction.py)

1. **Open the AWS Lambda Console:**
   - From the AWS Management Console, type **“Lambda”** in the search bar and select **“Lambda”**.

2. **Create a New Function:**
   - Click **“Create function”**.
   - Select **“Author from scratch”**.
   - Enter **Function name:** `UploadFunction`
   - Choose **Runtime:** `Python 3.x` (select the latest version available).
   - Under **Permissions**, choose **“Use an existing role”** and select **“LambdaS3AccessRole”**.
   - Click **“Create function”**.

3. **Copy and Paste Code:**
   - In the function code editor, replace any default code with the following code:

```python
import os
import boto3
import base64

s3 = boto3.client('s3')
BUCKET = os.environ['BUCKET_NAME']

def lambda_handler(event, context):
    # Retrieve file name from query string parameters
    params = event.get('queryStringParameters') or {}
    file_name = params.get('fileName')
    if not file_name:
        return {
            'statusCode': 400,
            'body': 'Missing fileName parameter'
        }
    
    # Decode body if base64 encoded (API Gateway sends binary files as base64 encoded)
    body = event.get('body')
    if event.get('isBase64Encoded', False):
        file_content = base64.b64decode(body)
    else:
        file_content = body.encode('utf-8')
    
    try:
        s3.put_object(Bucket=BUCKET, Key=file_name, Body=file_content)
    except Exception as e:
        return {
            'statusCode': 500,
            'body': f'Error uploading file: {str(e)}'
        }
    
    return {
        'statusCode': 200,
        'body': f'File "{file_name}" uploaded successfully'
    }
```

4. **Set Environment Variable:**
   - Scroll down to the **“Environment variables”** section.
   - Click **“Edit”**, then **“Add environment variable”**.
   - Set **Key:** `BUCKET_NAME` and **Value:** `my-multi-type-file-share-bucket` (or your chosen bucket name).
   - Click **“Save”**.

5. **Click “Deploy”** to save changes.

#### B. Download Function (DownloadFunction.py)

Repeat similar steps to create the download function:

1. **Create a New Function:**
   - In the Lambda console, click **“Create function”**.
   - Choose **“Author from scratch”**.
   - **Function name:** `DownloadFunction`
   - **Runtime:** `Python 3.x`
   - Under **Permissions**, choose **“Use an existing role”** and select **“LambdaS3AccessRole”**.
   - Click **“Create function”**.

2. **Copy and Paste Code:**
   - Replace the default code with:

```python
import os
import boto3
import base64

s3 = boto3.client('s3')
BUCKET = os.environ['BUCKET_NAME']

def lambda_handler(event, context):
    params = event.get('queryStringParameters') or {}
    file_name = params.get('fileName')
    if not file_name:
        return {
            'statusCode': 400,
            'body': 'Missing fileName parameter'
        }
    try:
        response = s3.get_object(Bucket=BUCKET, Key=file_name)
        file_content = response['Body'].read()
    except Exception as e:
        return {
            'statusCode': 500,
            'body': f'Error downloading file: {str(e)}'
        }
    
    return {
        'statusCode': 200,
        'isBase64Encoded': True,
        'headers': {
            'Content-Type': 'application/octet-stream',
            'Content-Disposition': f'attachment; filename="{file_name}"'
        },
        'body': base64.b64encode(file_content).decode('utf-8')
    }
```

3. **Set Environment Variable:**
   - Under **“Environment variables”**, add the key `BUCKET_NAME` with the value `my-multi-type-file-share-bucket`.
   - Click **“Save”** and then **“Deploy”**.

---

### Step 4.4: Configure API Gateway with Binary Support

1. **Create a New API:**
   - Open the **API Gateway Console** by typing **“API Gateway”** in the AWS Management Console search bar.
   - Click **“Create API”**.
   - Choose **“REST API”** (or HTTP API if preferred; this guide assumes REST API for clarity).
   - Select **“Build”** under the REST API option.

2. **Configure the API:**
   - **API Name:** Enter `my-file-sharing-api`
   - Click **“Create API”**.

3. **Set Binary Media Types:**
   - In your newly created API’s dashboard, in the left navigation pane, click on **“Settings”**.
   - In the **Binary Media Types** section, click **“Add Binary Media Type”**.
   - Enter `*/*` to support all binary types (or list specific types like `application/octet-stream`, `image/png`, etc.).
   - Click **“Save Changes”**.

4. **Create Resource and Methods:**
   - In the left navigation pane, click **“Resources”**.
   - Click the **“Actions”** drop-down and choose **“Create Resource”**.
   - In the **New Child Resource** window:
     - Set **Resource Name** as `files` and **Resource Path** as `/files`.
     - Click **“Create Resource”**.
   - Select the `/files` resource. Click **“Actions”** and then choose **“Create Method”**.
   - For the **POST method**:
     - Select **POST** from the drop-down, then click the checkmark.
     - In the **Setup** screen, select **Lambda Function** integration.
     - Ensure **Use Lambda Proxy integration** is checked.
     - Enter the Lambda function name: `UploadFunction`
     - Click **“Save”**. Confirm the role if prompted.
   - For the **GET method**:
     - With the `/files` resource still selected, click **“Actions”** and then **“Create Method”**.
     - Select **GET** and click the checkmark.
     - Select **Lambda Function** integration.
     - Check **Use Lambda Proxy integration**.
     - Enter the Lambda function name: `DownloadFunction`
     - Click **“Save”**.

5. **Deploy the API:**
   - In the **Actions** drop-down, click **“Deploy API”**.
   - In the **Deploy API** dialog:
     - For **Deployment stage**, select **[New Stage]**.
     - Enter **Stage Name** as `dev`.
     - Click **“Deploy”**.
   - Note the **Invoke URL** provided (you will need it for testing).

---

### Step 4.5: Deploy Your Lambdas and API

At this point, your Lambda functions and API Gateway configuration should be live. You can verify the following:
- The Lambda functions are deployed and have the correct environment variable set.
- The API Gateway has your `/files` resource with both **GET** and **POST** methods integrated with the correct Lambda functions.
- Binary media types are correctly configured.

---

### Step 4.6: Testing the Platform via Terminal (cURL)

#### A. Uploading a File

1. Open your terminal.
2. Run the following command (replace `<api-id>` and `<region>` with values from your Invoke URL, e.g., `abc123.execute-api.us-east-1.amazonaws.com`):

```bash
curl.exe --location "https://<api-id>.execute-api.<region>.amazonaws.com/dev/files?fileName=<filename>" ` 
--header "Content-Type: application/octet-stream" ` 
--data-binary "@<path-to-local-file>"
```

- **Explanation:**  
  - `--data-binary` sends the file in its original binary form.  
  - The `fileName` query parameter tells the Lambda function what to name the file in S3.

#### B. Downloading a File

1. In your terminal, run the following command:

```bash
curl.exe --location "https://<api-id>.execute-api.<region>.amazonaws.com/dev/files?fileName=<filename>" -o "<path-to-save-local-file>"
```

- **Explanation:**  
  - This command sends a GET request to the API.
  - The file is returned as a base64-encoded response by the Lambda, which API Gateway decodes based on your settings.
  - The `--o` flag saves the file locally as `<path-to-save-local-file>`. 

---

## 5. Final Notes and Best Practices

- **API Gateway Binary Settings:**  
  Ensure that the binary media types include your file formats (or use `*/*` for all types). If files are not handled correctly, verify this setting in the API Gateway’s **Settings** tab.

- **Error Handling and Logging:**  
  Enhance error handling in your Lambda functions. Use AWS CloudWatch to monitor logs for troubleshooting.

- **Security Considerations:**  
  Secure your API endpoints using API keys, IAM authorizers, or Lambda authorizers. For production, consider tighter S3 bucket policies and IAM restrictions.

- **Future Enhancements:**  
  Consider adding features like file deletion, versioning, user authentication, and automated deployments (using AWS SAM, the Serverless Framework, or CloudFormation).

---

## 6. Conclusion

By following this detailed report, you now have the step-by-step instructions—including which tabs to click, what settings to configure, and how to deploy—to build and deploy a serverless file sharing platform on AWS that supports multiple file types. This solution leverages Amazon S3’s scalability and durability, AWS Lambda’s ease of use, and Amazon API Gateway’s flexibility in handling binary data.
