# Migrating to AWS: EC2 and S3 Integration

This guide outlines the step-by-step changes we made to our House Price Prediction application to prepare it for a production-grade deployment on AWS, specifically utilizing **EC2** for computing and **S3** for model storage.

## Why are we doing this?
In a local environment, it's easy to store large Machine Learning models (`.pkl` files) directly in your project folder. However, in a production environment:
1. Large files bloat the Git repository and make deployments slower.
2. If we want to retrain the model and deploy a new version, we would have to redeploy the entire application code. 
3. By moving the models to an S3 bucket, our EC2 instance can dynamically fetch the latest model at runtime.

---

## Step 1: Updating Python Dependencies (`requirements.txt`)

To interact with AWS services from Python, we need the official AWS SDK, called `boto3`.

**What we did:**
We added `boto3` to our `requirements.txt` file.

```diff
  scikit-learn==1.2.2
  catboost
  gunicorn
+ boto3
```

When we set up our EC2 instance, running `pip install -r requirements.txt` will automatically install the AWS SDK.

---

## Step 2: Refactoring `app.py` for Dynamic S3 Loading

Previously, our application loaded the Machine Learning model and scaler locally from the disk using `pickle`. We modified `app.py` to fetch these files directly from an S3 bucket over the network.

**What we did:**

1. **Imported Required Libraries:** Added imports for `boto3`, `os`, and specific error handling `NoCredentialsError`.
2. **Configured the S3 Bucket:** Added an environment variable `S3_BUCKET_NAME` to store the name of the bucket. This allows us to change the bucket name in the EC2 environment without modifying the code.
3. **Created a Fetch Function:** Wrote a helper function `load_from_s3(key)` that:
   - Connects to AWS S3 using `boto3.client('s3')`.
   - Downloads the requested file (model or scaler) directly into memory.
   - Deserializes the downloaded bytes using `pickle.loads()`.

### Code Breakdown:

```python
import os
import boto3
from botocore.exceptions import NoCredentialsError

# Fetch the S3 Bucket name from environment variables
S3_BUCKET_NAME = os.environ.get('S3_BUCKET_NAME', 'your-s3-bucket-name-here')

# Initialize the AWS S3 Client
s3_client = boto3.client('s3')

def load_from_s3(key):
    print(f"Loading {key} from S3 bucket: {S3_BUCKET_NAME}...")
    try:
        # Fetch the object from S3
        response = s3_client.get_object(Bucket=S3_BUCKET_NAME, Key=key)
        
        # Read the binary data and load it using pickle
        return pickle.loads(response['Body'].read())
        
    except NoCredentialsError:
        print("Error: AWS credentials not found. Attach an IAM role to your EC2 instance.")
        raise
    except Exception as e:
        print(f"Error fetching {key} from S3: {str(e)}")
        raise

# Dynamically load models at app startup
model = load_from_s3('housepred.pkl')
scaler = load_from_s3('scaler.pkl')
```

---

## Step 3: AWS Setup Checklist (To be done on the AWS Console)

Now that the code is ready, here is the architecture setup you will perform in your AWS Console to make this work:

1. **Create an S3 Bucket:**
   - Go to Amazon S3.
   - Create a bucket (e.g., `my-house-pricing-models`).
   - Upload your `housepred.pkl` and `scaler.pkl` files into this bucket.

2. **Create an IAM Role:**
   - Go to IAM (Identity and Access Management).
   - Create a new Role for an **EC2 Service**.
   - Attach the policy: `AmazonS3ReadOnlyAccess`.
   - Name the role something like `EC2-S3-Model-Reader`.

3. **Launch an EC2 Instance:**
   - Launch an Ubuntu EC2 instance.
   - **Crucial Step:** Under "Advanced Details" during setup, assign the IAM Role you created (`EC2-S3-Model-Reader`) to the instance. This grants the server permission to read from S3 securely without hardcoding secret API keys in our Python code.

4. **Deploy the App to EC2:**
   - SSH into the instance.
   - Clone your project.
   - Export the bucket name: `export S3_BUCKET_NAME="your-bucket-name"`
   - Install dependencies and run the Flask application.

Because of the IAM Role, `boto3` will securely and automatically retrieve temporary credentials in the background, keeping our architecture secure and production-ready!
