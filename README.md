From Upload to Alert: Auto-Creating S3 Buckets with Timestamps Using Lambda & SNS.
==================================================================================

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*sxvdO1urQfbMEAg-GC68JQ.png)

[Reference](https://medium.com/@nikhilsiri2003/from-upload-to-alert-auto-creating-s3-buckets-with-timestamps-using-lambda-sns-9108502ccd58)

by [Nikhil Raj A](https://medium.com/@nikhilsiri2003?source=post_page---byline--9108502ccd58---------------------------------------)






Introduction
------------

Amazon S3 (Simple Storage Service) is a powerful cloud storage solution provided by AWS that allows users to store and retrieve any amount of data at any time. In many real-world scenarios, organizations require automation to manage data more effectively. One such use case is the automated creation of new S3 buckets with timestamps whenever a new object is uploaded to an existing bucket. This approach not only helps in organizing data chronologically but also enables better tracking and archiving of uploads.

To further enhance this automation, integrating email notifications ensures that users or administrators are immediately alerted when an upload occurs. This is accomplished by leveraging AWS services such as **S3 Event Notifications**, **Lambda Functions**, **SNS (Simple Notification Service)**, and **CloudWatch**. The Lambda function responds to upload events in the source bucket, dynamically creates a new bucket with a timestamped name, and triggers an SNS topic to send an email alert.

![Simple Workflow of Procedure](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*S4WWQKtDSZpcxflbzbKQ6w.png)

**What is Simple Storage Service (S3)?**
----------------------------------------

Amazon **Simple Storage Service** (Amazon S3) is a scalable, high-speed, web-based cloud storage service designed to store and retrieve any amount of data at any time. Launched by Amazon Web Services (AWS), S3 is widely used for a variety of purposes, including backup and restore, content distribution, disaster recovery, and data archiving.

What is Simple Notification Service(SNS)?
-----------------------------------------

Amazon **Simple Notification Service (SNS)** is a fully managed messaging service provided by AWS that allows you to send notifications to a large number of recipients or systems. It provides a highly scalable, reliable, and cost-effective solution for sending messages, alerts, and notifications to end-users or other applications. SNS enables the creation and management of **topics**, to which subscribers can subscribe to receive messages.

What is Lambda ?
----------------

**AWS Lambda** is a fully managed serverless computing service provided by Amazon Web Services (AWS) that lets you run your code in response to events without provisioning or managing servers. With Lambda, you can write code to process events such as file uploads, database changes, or HTTP requests, and AWS automatically manages the infrastructure required to run that code.

Prerequisites
=============

To get started with this project , you need the following:

*   AWS Account
*   Gmail Account

Step 1 : Go to S3 and create a S3 Bucket
----------------------------------------

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*lJfPlUqbaPKDpu2ewn_0ag.png)

1.  Select **General Purpose** as Bucket-type.
2.  Provide a name for the S3 bucket.

![creating a S3 bucket](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*kx6ysr5a4WGlZZs1cJ7c3w.png)

3. Untick the Block all public access, and also enable **bucket-versioning .**

4. Enable Bucket-key and Click **create bucket .** Then a bucket will be created under the name you have provided .

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*O5VLiGmhgTQWGrAIxpUuDQ.png)

**Step 2 : Provide the bucket policy of the S3 bucket**
-------------------------------------------------------

1.  Once the S3 bucket is created , Go to **Permissions** to provide the S3 bucket policy .

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*NHRCkNtqsSWTWRhpGLdcJQ.png)

2. Under **Permissions** you will find **bucket-policy ,** click on **Edit** to provide the bucket policy. The policy must be written using JSON.

```
{
  "Version": "2012-10-17",
  "Statement": [    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::bucket-genie/*"
    }
  ]
}
```

3. Then after the policy is given inside the bucket policy , click on **save changes** to update the changes in the S3 bucket.

**Step 3 : Go to Lambda and create a Lambda function**
------------------------------------------------------

![creating lambda function](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*eZqXFYhrTvZeh344_UQ7mQ.png)

1.  Click on **“create a function”** , Select an option called “**author** **from Scratch”.**
2.  Provide a name for your function.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*jF_DFc_dK49ZSli0-xQ3kg.png)

3. Select **python 3.13** as lambda runtime , and **x86_64** as the **Architecture.**

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*gxYtBHFrOEMD8G1AOzE09w.png)

4. After selecting the runtime and architecture , click on **create function** a lambda function will be created.

5. Once the function is created you can see a image like this with the function name on the top.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*mwpo_QgCaGI4khhx9Pkrlg.png)

6. Under code section provide the Python code which is the main key for the project .

```
import boto3
from datetime import datetime
import urllib.parse
s3 = boto3.client('s3', region_name='ap-south-1')
sns = boto3.client('sns', region_name='ap-south-1')
# Fixed bucket name with timestamp - generated only once
BUCKET_NAME = 'bucket-genie-backup-20250509'
REGION = 'ap-south-1'
SNS_TOPIC_ARN = 'arn:aws:sns:ap-south-1:530424100396:S3'
def bucket_exists(bucket_name):
    try:
        s3.head_bucket(Bucket=bucket_name)
        return True
    except s3.exceptions.ClientError:
        return False
def create_bucket(bucket_name):
    s3.create_bucket(
        Bucket=bucket_name,
        CreateBucketConfiguration={'LocationConstraint': REGION}
    )
    print(f" Created bucket: {bucket_name}")
def copy_to_bucket(source_bucket, object_key, destination_bucket):
    copy_source = {'Bucket': source_bucket, 'Key': object_key}
    s3.copy_object(
        Bucket=destination_bucket,
        CopySource=copy_source,
        Key=object_key
    )
    print(f" Copied {object_key} from {source_bucket} to {destination_bucket}")
def send_sns_notification(bucket_name, created_time):
    message = f"""Hello user,
This is to inform you that a new S3 bucket has been created for backup purposes.
🔹 Bucket Name: {bucket_name}
🔹 Region: {REGION}
🔹 Created At (UTC): {created_time}
All future uploads will be backed up to this bucket.
Thank You,
AWS Lambda Backup System
"""
    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject=' S3 Backup Bucket Created',
        Message=message
    )
    print("📧 Email notification sent via SNS.")
def lambda_handler(event, context):
    source_bucket = event['Records'][0]['s3']['bucket']['name']
    object_key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'])
    try:
        is_new = False
        if not bucket_exists(BUCKET_NAME):
            create_bucket(BUCKET_NAME)
            is_new = True
        copy_to_bucket(source_bucket, object_key, BUCKET_NAME)
        if is_new:
            created_time = datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S')
            send_sns_notification(BUCKET_NAME, created_time)
        return {
            'statusCode': 200,
            'body': f"Backup complete. Object '{object_key}' copied to '{BUCKET_NAME}'."
        }
    except Exception as e:
        print(f" Error: {str(e)}")
        raise e
```

7. After adding the code you need to test and deploy the code to make sure the code is running perfectly without any errors .

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*yUQPamTBMgDvY0mp8FahjA.png)

8. Click on **Add trigger** , so that whenever an object is been uploaded into the S3 bucket then it automatically triggers the lambda function.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*nplGjS_6iWE9yPLxUJf7IQ.png)

9. Select the **Source** as S3 , under bucket section select the bucket that you have created . Then click on **Add** to add the trigger into lambda function.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*1M0VdY0Nfnf3XYd93CYYIg.png)

**Step 4 : Go to IAM Roles and Attach the policies.**
-----------------------------------------------------

1.  Under lambda function you can see **permissions ,** click on the role name shown below the role name which will redirect to IAM roles and policies .

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*CHzZHB5ROHsZIued6HA_xQ.png)

2. You can see **lambda-genie-role-4ttdny90** is the role name , click on **Add Permissions** to attach the policies.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Eufji5BJ68dsxAkpCG9ZQA.png)

3. Search for AmazonS3FullAccess and once you find the policy , select that policy and click on **Add permission.**

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Wikglsbh3JCCO3oHjwRatw.png)

**Step 5 : Go to SNS and create a SNS topic for Notification**
--------------------------------------------------------------

![creating SNS topic](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Jpu4iAlRaVwju5X7FtpALw.png)

1.  click on **next step** to create a topic , select the topic type as **standard .**
2.  provide the name and Display name for the topic so that when you get notified , it shows by the display name.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*UW6mTsGDlEYSSv9rlJkeVQ.png)

3. Then click on **create topic** to create topic . And once the topic is been create you need to create **subscription** because the SNS itself doesn’t know the destination point, so we give subscription as **Email , SMS , HTTP/HTTPS endpoint or SQS .**

4. while creating subscription under protocol, select the type of platform you need to the message to be delivered . In my case i have selected the **email ,** so the message will be delivered to my Gmail.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*IKj0NIgp2iEAEdCIMB5tXw.png)

5. Click on **create subscription** after giving all the details . After creating the subscription a mail would be delivered to your Gmail by which you have registered for the subscription . You need to confirm the subscription by click on it .

![subscription confirmed](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*eoKewTJnwIsab1HKtsMaOQ.png)

6. After the subscription is been confirmed , you’ll need to click on **publish message** for publishing the desired message to the user.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*nnqzWgBUuh9YDmT6O1Wfhw.png)

7. Provide the subject if needed , then go to message body and type the desired the message that you want to publish . Then click on **publish Message.** Remember that **publish message** is used for publishing the message manually.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*dQqqGGdojF_4SvhviYtUQw.png)

**step 6 : Upload some files into your S3 bucket.**
---------------------------------------------------

1.  Go to your s3 bucket and upload some files or objects into it .

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Jn56i_Mw0iEFwF4Ibr1OPg.png)

2. After uploading of the files and objects , **click on upload .** After clicking on upload you should see a new s3 bucket created with **timestamp.**

![New S3 bucket created](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*OpRVpvQqk24vfHAI5B1O5Q.png)

3. Let me Breakdown the newly created S3-bucket name.

![Breakdown of the newly created S3 — Bucket](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*qWm8QOTHpI3gh-1SPbIqnQ.png)

4. Now go to Gmail , where you will be notified that a new s3 bucket as been created . And this is the final result .

![Email Notification of the Newly created bucket](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*q31hyHTHK7vRQTI0YCDBUQ.png)

Conclusion
----------

When a file is uploaded to the `bucket-genie`, a Lambda function creates a new timestamped bucket, copies the file, and sends an email notification via SNS. It demonstrates automated event handling, object replication, and alerting in a scalable cloud setup. This approach is useful for real-time monitoring, backups, and alert-driven applications.
