### 🎥 Demo Video
Watch the full crawler in action on LinkedIn:  
👉 [Watch Demo Video](https://www.linkedin.com/embed/feed/update/urn:li:ugcPost:7350978672381616128?collapsed=1)

# 📓 Build Journal: Serverless Web Crawler

This build journal documents the full technical journey of designing and deploying a Serverless Web Crawler on AWS. It captures the architecture, challenges, iterations, and lessons learned during development.

## Diagram

<img src="screenshots/0-diagram.png" width="750">

## 🛠️ Step 1: Preparing a Website to Crawl
As the first step, I needed a website to test the crawler.
I already owned a domain and had an existing static webpage (a simple React app) hosted in S3.
To make the crawler more robust, I build a new React app with React Router for client-side dynamic navigation and upload it to S3.

## React App Setup:
### 1. Create React app:
``` bash 
npx create-react-app demo-app
cd demo-app
npm install react-router-dom
```
### 2. Create JS pages in /src:
- Home.js (links to /pictures, /tutorial, /contact)
- Tutorial.js (links to /tutorial/courses)
- Courses.js (links to /contact)
- Other pages: /pictures, /contact

### 3. Configure routing in App.js.

<img src="screenshots/20-react app router dom and LINK TO.png" width="750">

### 4. Build and upload to S3:
``` bash
npm run build
```
### 5. Clear S3 bucket and upload the new build.

### 6. Create CloudFront invalidation (/*) to refresh the cache.
- First attempt failed (white page).
- Fixed by clearing browser cache and issuing another invalidation.

✅ **Result: Static site deployed and ready for testing.**

<img src="screenshots/2- domain loading correct webpage through cloudfront.png" width="750">

## Step 2: Web Crawler Functionality
The architecture revolves around two Lambda functions and AWS services for orchestration:

### 1. Initiator Lambda (Python)
- Receives a root URL.
- Saves URL in DynamoDB with a unique runID.
- Sends URL to an SQS Queue for processing.

### 2. Crawler Lambda (Node.js)
- Triggered by SQS messages.
- Loads pages using Puppeteer (headless Chromium).
- Extracts all `<a href=""> and <Link to="">` links (for React Router).
- Checks DynamoDB for visited URLs.
- Enqueues new URLs into SQS for recursive crawling.

### 3. Process continues until all unique
- internal links are visited, respecting a max crawl depth.

## Step 3: AWS Infrastructure
### ✅ Initiator Lambda
- Runtime: Python 3.12
- Purpose: Starts the crawl process by writing the root URL into DynamoDB and queuing it in SQS.

  ***Execution Role:***
  - Created a new IAM role.
  - Added DynamoDB permissions: "BatchGetItem", "BatchWriteItem", "PutItem", "DeleteItem", "GetItem", "Scan", "Query", "UpdateItem".
  - Added SQS permission: "SendMessage" for the crawler SQS queue only.
  ***Creation Process:***
  - Navigated to Lambda > Create Function.
  - Chose runtime Python 3.12.
  - Selected “Create a new role” and attached “Simple microservice permissions” from AWS templates.

<img src="screenshots/3-Lambda functions created.png" width="750">

### ✅ Crawler Lambda
- Runtime: Node.js
- Purpose: Consumes messages from SQS, uses Puppeteer (headless Chromium) to load pages, extracts links, checks visited URLs in DynamoDB, and queues new links.

  **Execution Role:**
  - Created a new IAM role.
  - Added DynamoDB permissions: "BatchGetItem", "BatchWriteItem", "PutItem", "DeleteItem", "GetItem", "Scan", "Query", "UpdateItem".
  - Added SQS permissions: "SendMessage", "ReceiveMessage", "DeleteMessage", "GetQueueAttributes", "GetQueueUrl" for both primary SQS and DLQ.
  **Creation Process:**
  - Navigated to Lambda > Create Function.
  - Chose runtime Node.js.
  - Selected “Create a new role” and attached “SQS Poller role” and “Simple microservice permissions” from AWS templates.

<img src="screenshots/6.2-eddited IAM policy for Crawler Lambda role.png" width="750">


### ✅ DynamoDB Table
- Table Name: VisitedURLs
- Partition Key: url
- Sort Key: runID
- Capacity Mode: On-Demand (pay-per-request).

<img src="screenshots/4-DynamoDB.png" width="750">

### ✅ SQS and DLQ Setup
- Primary Queue: For passing URLs to the Crawler Lambda.
- Dead Letter Queue (DLQ): Configured to capture failed messages for debugging and recovery.
  **Created two queues:**
    - Primary SQS Queue (no encryption for this case).
    - DLQ and associated it with the primary SQS.

<img src="screenshots/30-SQS messages working.png" width="750">

### ✅ Linking SQS to Lambda
- Added SQS trigger to the Crawler Lambda.
- Set batch size to 1 for fine-grained control.
- Ensured the Crawler Lambda has permission to poll messages from SQS by updating its IAM policy.

<img src="screenshots/3-Crawler Lambda.png" width="750">

### ✅ Concurrency & Error Handling
- Set reserved concurrency for the Crawler Lambda to 2 to control parallelism and avoid resource exhaustion.
  **Configured asynchronous invocation with DLQ:**
  - Enabled retry and fallback logic in case of processing failures.
  - Note: This was not to prevent runaway crawls but to capture errors during crawling for later investigation.

<img src="screenshots/6.3-Crawler Lambda Asynchronous invocation.png" width="750">

### 🛡 Domain Scope Restriction Logic
Implemented domain filtering logic in the Crawler Lambda code to ensure URLs outside the target domain are ignored.
- Example: Prevented the crawler from accidentally queuing links to sites like YouTube or Facebook.
- This restriction happens at the application level, not at the infrastructure level.

## ❌ Wrong Approach: Python Crawler

Initially, I attempted to build the web crawler using Python, leveraging Selenium and a custom headless Chromium layer. The goal was to handle both static and dynamic content. While this approach worked locally, it ultimately failed to deploy on AWS Lambda due to several technical challenges.

### 🛠 Steps Taken

1. **Dependencies Setup**
- Created a requirements.txt file with necessary Python packages.
- Installed dependencies locally using:
``` bash
pip install -r requirements.txt --target ./dependencies
```

- Organized directories:
  - /dependencies
  - /initiator_lambda
  - /crawler_lambda

- Zipped each directory individually for upload to S3.

2. **Lambda Layer Creation**
- Uploaded the zipped dependencies folder to S3.
- Created a Lambda Layer from S3 URL with runtime set to Python 3.12.
- Attached the layer to both Initiator and Crawler Lambda functions.

3. **Code Deployment**
- Uploaded zipped Python code for Initiator (initiator.handle) and Crawler (crawler.handle) Lambda functions.
- Set handler paths in Lambda runtime settings.

4. **Initial Testing and Issues**
- Testing the Initiator Lambda worked after adjusting IAM policies (added sqs:GetQueueUrl).
- However, Crawler Lambda failed with:
``` bash
ModuleNotFoundError: No module named 'requests_html'
```
- Verified that the module existed in the layer, but Lambda runtime couldn’t locate it.

###  Challenges Faced

1. **Python Dependency Packaging**
- Initially packaged dependencies for Python 3.11 locally, but Lambda runtime was Python 3.12.
- Fixed this by rebuilding dependencies in a Linux environment to match AWS runtime.

2. **Incorrect Layer Directory Structure**
- AWS Lambda requires Python layers to follow this structure:
``` bash
   layer_content.zip
    └ python
       └ lib
         └ python3.12
              └ site-packages
                  └ requests
                  └ <other dependencies>
```
- Repackaged layers following this format and reuploaded.

3. **Additional Dependency Errors**
- Encountered issues with lxml and related modules.
- Added lxml_html_clean to requirements.txt to resolve dynamic imports.

4. **Headless Browser Challenges**
- Attempted to use a custom Chromium layer for headless browser automation.
- Local testing succeeded, but Lambda deployments repeatedly hit the 250MB size limit for layers.

5. **Dynamic Content Rendering**
- Realized React apps using react-router-dom don’t expose `<a href="...">` links in the initial HTML.
- Selenium could handle this locally but failed to scale in Lambda due to size and runtime constraints.

### ⚠️ Outcome

While the Python crawler worked locally, it could not be deployed to AWS Lambda due to:
- Lambda layer size constraints (250MB).
- Complexity of Python dependency packaging for Lambda runtime.
- Dynamic React Router links not being detected without fully rendering the page.

This approach was abandoned in favor of a JavaScript-based crawler using Puppeteer-Core and Sparticuz Chromium, which overcame these limitations.

<img src="screenshots/21- code tested locally and its working.png" width="750">

## ✅ Correct Approach: JavaScript Crawler (Final Implementation)

I rewrote the crawler in JavaScript to overcome Python’s Lambda limitations and improve dynamic content handling. This approach uses Puppeteer-Core with Sparticuz Chromium—a Chromium build optimized for AWS Lambda.

###  Why Switch to JavaScript?

  - Python + Selenium Limitations: Hit Lambda layer size limit (250MB). Couldn’t package Chromium/Selenium effectively.

  - Sparticuz Chromium: Pre-optimized for AWS Lambda environments,  allowing headless browser automation within size constraints.

  - Dynamic Content Support: Puppeteer handles React Router links and dynamically rendered pages out of the box.

### Dependencies

Installed in a Linux environment to match AWS Lambda:
``` bash
   "@aws-sdk/client-dynamodb": "^3.687.0",
   "@aws-sdk/client-sqs": "^3.687.0",
   "follow-redirects": "^1.15.9",
   "puppeteer-core": "^23.7.1",
   "tar-fs": "^3.0.6",
   "uuid": "^11.0.2"
```
✅ **Lambda Layers Created**
- Layer 1: Sparticuz Chromium
``` bash
  Sparticuz Chromium v130.0.0
```
- Layer 2: Node.js dependencies (zipped from node_modules)

### Key Improvements

- Asset Exclusion for Performance: Excluded unnecessary content like images, videos, fonts, and stylesheets in Puppeteer to reduce page load time.

- Concurrency Control: Configured Lambda reserved concurrency (20) for parallel crawling. Each Lambda instance now handles separate URLs efficiently.

- Dynamic Content Handling: Added delays and scrolling logic to allow React Router-based pages to fully render before scraping links.

## 🪲 Debugging & Challenges

### 1. Duplicate Crawling
Initially, URLs were revisited endlessly. Fixed by checking DynamoDB’s VisitedURLs table before enqueuing new links.

### 2. Crawling Beyond Domain Scope
- Accidentally crawled external domains (e.g., YouTube, Facebook).
- Added domain scope restriction to prevent this.

### 3. Lambda Timeout Issues
- Handled cold starts and dynamic rendering delays.
- Increased timeout and tuned Puppeteer waits for dynamic routes.

### 4. AWS Client Connection Problems
- Adjusted SDK usage for compatibility with Node.js version on Lambda.

### 5. Layer Packaging Issues
Ensured layers were structured correctly for Lambda:
``` bash
   layer.zip
   └ nodejs
       └ node_modules
           └ <dependencies>
```
## Optimization: Lambda Power Tuning
Used AWS Lambda Power Tuning (Step Functions) to find the optimal memory and cost configuration.

### Power Tuning Details:
  - Power Values Tested: [128, 256, 512, 1024, 2048, 3008]
  - Best Memory Setting: 812MB
  - Reserved Concurrency: 20
  - Result: Reduced crawl time from ~60s → ~12s per URL

<img src="screenshots/34.5- power tuning charts 512-1024 range searching for cost optimization.png" width="750">

##  Final Deployment

- Environment Variables: Configured REGION, SQS_URL, MAX_DEPTH, and TIMEOUT for dynamic parameter control.

<img src="screenshots/35.2- ENV .png" width="750">

### Lambda Aliases:
  PROD: Points to stable release.
  TEST: Used for A/B testing new versions.

## ✅ Results

Successfully crawled:
  - My domain: cloudnecessities.com

<img src="screenshots/31.0- Crawling my website cloudneccesities.com.png" width="750">

  - Test site: drugastrana.rs

<img src="screenshots/31.5-DynamoDB - End of Crawl.png" width="750">

Both runs extracted all intended links dynamically.

### 🎥 Demo Video
Watch the full crawler in action on LinkedIn:  
👉 [Watch Demo Video](https://www.linkedin.com/embed/feed/update/urn:li:ugcPost:7350978672381616128?collapsed=1)

### Do you want to see all screenshots from project? 
👉 [All screenshots](docs/screenshots/)

## 📊 CloudWatch metrics show improved cost and performance after optimizations.
<img src="screenshots/33.8-Crawler Lambda - monitoring , after lambda optimization - cloudwatch lambda insights.png" width="750">


### Lessons Learned
Python vs JavaScript on Lambda: Puppeteer + Sparticuz Chromium works far better than Selenium in Lambda.

Layer Packaging: Critical to match AWS runtime environments.

Debugging: CloudWatch logs and Lambda Insights were essential for performance tuning.

### 🎥 Demo Video
Watch the full crawler in action on LinkedIn:  
👉 [Watch Demo Video](https://www.linkedin.com/embed/feed/update/urn:li:ugcPost:7350978672381616128?collapsed=1)

### Do you want to see all screenshots from project? 
👉 [All screenshots](docs/screenshots/)

## 🧑‍💻 Author
👋 Milos Faktor 💼 [LinkedIn](https://www.linkedin.com/in/milos-faktor-78b429255/)