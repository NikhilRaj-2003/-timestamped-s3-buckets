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
    print(f"✅ Created bucket: {bucket_name}")
def copy_to_bucket(source_bucket, object_key, destination_bucket):
    copy_source = {'Bucket': source_bucket, 'Key': object_key}
    s3.copy_object(
        Bucket=destination_bucket,
        CopySource=copy_source,
        Key=object_key
    )
    print(f"✅ Copied {object_key} from {source_bucket} to {destination_bucket}")
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
        Subject='✅ S3 Backup Bucket Created',
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
        print(f"❌ Error: {str(e)}")
        raise e
