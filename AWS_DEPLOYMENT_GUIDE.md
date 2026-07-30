# AWS Deployment Guide (Production Version - V3.0)

This guide provides step-by-step instructions for deploying the **House Price Prediction** Machine Learning Flask application to AWS. This production-grade architecture utilizes **Amazon S3** for dynamic model storage and an **EC2** instance for computing, secured via **IAM Roles**.

## Step 1: Create an S3 Bucket and Upload Models
To decouple our large `.pkl` files from the codebase, we store them in S3.
1. Log in to your [AWS Management Console](https://aws.amazon.com/console/).
2. Search for **S3** and open the S3 Dashboard.
3. Click **Create bucket**.
4. Name your bucket (e.g., `house-price-models-akansana`). It must be globally unique. Keep the default settings and click **Create bucket**.
5. Open your new bucket, click **Upload**, and upload `housepred.pkl` and `scaler.pkl` from your local machine.

## Step 2: Create an IAM Role for EC2
We need to grant our EC2 instance permission to read the files from S3 without hardcoding any AWS credentials in our code.
1. Search for **IAM** in the AWS Console.
2. Click **Roles** in the left sidebar, then click **Create role**.
3. Under **Trusted entity type**, select **AWS service**.
4. Under **Service or use case**, select **EC2** and click Next.
5. In the permissions search bar, type `AmazonS3ReadOnlyAccess`. Check the box next to it and click Next.
6. Name your role (e.g., `EC2-S3-Model-Reader`) and click **Create role**.

## Step 3: Launch an EC2 Instance
1. Go to the **EC2 Dashboard** and click **Launch instance**.
2. **Name:** Give your instance a name (e.g., `House-Price-App-V3`).
3. **OS Images (AMI):** Select **Ubuntu** (the default "Ubuntu Server 24.04 LTS" or similar). Make sure it says *Free tier eligible*.
4. **Instance Type:** Keep it as **t2.micro** or **t3.micro** (*Free tier eligible*).
5. **Key Pair:** 
   - Click **Create new key pair**.
   - Name it (e.g., `house-app-key`), choose **RSA** and **.pem**, then **Create key pair**.
6. **Network Settings:**
   - Check **Allow SSH traffic from Anywhere**.
   - Check **Allow HTTP traffic from the internet**.
7. **Advanced Details (CRITICAL):**
   - Scroll down to the **Advanced details** section.
   - Under **IAM instance profile**, select the role you created in Step 2 (`EC2-S3-Model-Reader`).
8. Click **Launch instance**.

## Step 4: Open Port 8000 for Gunicorn
1. Go to your EC2 Instances list, select your new instance.
2. Under the **Security** tab, click the **Security group** link.
3. Click **Edit inbound rules** -> **Add rule**.
4. Set **Type:** Custom TCP, **Port range:** 8000, **Source:** Anywhere-IPv4 (`0.0.0.0/0`).
5. Click **Save rules**.

## Step 5: Connect to Your EC2 Server
1. Open your terminal.
2. Navigate to where your `.pem` key was downloaded.
3. Fix permissions on the key (Mac/Linux only):
   ```bash
   chmod 400 house-app-key.pem
   ```
4. SSH into the server (replace the IP with your EC2's "Public IPv4 address"):
   ```bash
   ssh -i "house-app-key.pem" ubuntu@<YOUR-EC2-PUBLIC-IP>
   ```

## Step 6: Prepare the Server & Install Miniconda
1. Update the server and install tools:
   ```bash
   sudo apt update
   sudo apt upgrade -y
   sudo apt install git wget build-essential -y
   ```
2. Install Miniconda for a stable Python 3.10 environment:
   ```bash
   wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
   bash Miniconda3-latest-Linux-x86_64.sh -b -p $HOME/miniconda
   source $HOME/miniconda/bin/activate
   conda init
   ```
3. Refresh your shell by typing `source ~/.bashrc` or logging out and back in.

## Step 7: Download the Project
Clone the V3.0 repository:
```bash
git clone https://github.com/akansana-work/House-Price_V3.0.git
cd House-Price_V3.0
```

## Step 8: Set up the ML Python Environment
Create the Conda environment to ensure Scikit-Learn, NumPy, and Boto3 work without conflicts:
```bash
conda create --override-channels -c conda-forge -n ml_env python=3.10 -y
conda activate ml_env
pip install -r requirements.txt
```

## Step 9: Run the Application
You must export the S3 bucket name so the Flask app knows where to fetch the `.pkl` files. Replace `your-bucket-name` with the bucket you created in Step 1.

```bash
export S3_BUCKET_NAME="house-price-models-akansana"
gunicorn --bind 0.0.0.0:8000 app:app
```

## Step 10: View Your Live App!
Open your web browser and go to:
```
http://<YOUR-EC2-PUBLIC-IP>:8000
```
Because of the IAM Role attached to the EC2 instance, the app will seamlessly authenticate with AWS and pull the machine learning models down into memory when it starts up!

## Optional: Keep it Running in the Background
To keep the application running after closing your terminal:
1. Press `Ctrl + C` to stop the current server.
2. Run:
   ```bash
   export S3_BUCKET_NAME="house-price-models-akansana"
   nohup gunicorn --bind 0.0.0.0:8000 app:app &
   ```
3. Safely close your terminal.
