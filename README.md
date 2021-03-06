# Upload-VM-to-AWS-EC2
This repo describe the step to upload a VM to AWS EC2

## Pre-requirements
* You will need to install the AWS CLI. Configure it to have access to your account.
* Create a S3 bucket on your AWS account

## Prepare the file
### Export the VM
 If you're using the Oracle VM VirtualBox:
 * Right click on your VM, Export to OCI
 * For format, choose Open Virtualization Format 2.0
 * Give the file a name
 * Strip all network adapter MAC addresses
 * Incluse ISO image files
 * Next

### Split the file
Because the file is too big (AWS CLI currently supports files up to 5GB) we will need to use the multipart upload. For that we need to split the file. Skip this tutorial and upload it into one chuck if your file is smaller than 5G.
To split the file, on Linux, run:
`split --bytes=1G -d --suffix-length=2 VM-FILE.ova VM-FILE-PART-NAME.ova. --verbose`
* `--bytes` is the size of the chunks
* `-d` uses numbers as suffix
*  `--suffix-length` uses two characters size
*  `VM-FILE.ova` is the file name
*  `VM-FILE-PART-NAME.ova.` is the prefix of the create part files (VM-FILE-PART-NAME.ova.01, VM-FILE-PART-NAME.ova.02, VM-FILE-PART-NAME.ova.03..)
Read more about Split here: [split invocation]

### Upload to S3
The steps are:
1. Create a multipart file
2. Upload each part
3. Complete the upload

Based on this link: [s3 multipart upload]

* In order to create the multipart file, run:
`aws s3api create-multipart-upload --bucket BUCKET-NAME --key FINAL-REJOINED-FILE-KEY`
The output should be:
```
{
    "Bucket": "BUCKET-NAME",
    "Key": "FINAL-REJOINED-FILE-KEY",
    "UploadId": "A HUGE RANDOM NUMBER, LETTERS AND SYMBOLS ID"
}
```

* Then we upload the parts:
`aws s3api upload-part --bucket BUCKET-NAME --key FINAL-REJOINED-FILE-KEY --upload-id "THAT BIG ID FROM THE PREVIOUS COMMAND" --body THE-PART-FILE.00 --part-number 1`
So, for each file upload, increase the `--part-number`.
The output should be an ETag.

* Then complete the multipart upload
`aws s3api complete-multipart-upload --multipart-upload file://mpustruct --bucket BUCKET-NAME --key 'FINAL-REJOINED-FILE-KEY' --profile personal --upload-id THAT BIG ID FROM THE PREVIOUS COMMAND`
The `--multipart-upload file://mpustruct` points to a file in your local folder that contains the multipart upload struct in the following format:
```
{
  "Parts": [
    {
      "ETag": "THE ETAG FROM upload-part PART-NUMBER 1",
      "PartNumber": 1
    },
    {
      "ETag": "THE ETAG FROM upload-part PART-NUMBER 2",
      "PartNumber": 2
    },
    ...
  ]
}
```

## Create an image
### Create role and policy
To import an image into AMI you will need a role named `vmimport` and a policy to allow that role to import/export images.

1. To create the role, create a file named `trust-policy.json` into your local with the following content:
```
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Principal": { "Service": "vmie.amazonaws.com" },
         "Action": "sts:AssumeRole",
         "Condition": {
            "StringEquals":{
               "sts:Externalid": "vmimport"
            }
         }
      }
   ]
}
```
Then create the role running the following command:
`aws iam create-role --role-name vmimport --assume-role-policy-document "file://trust-policy.json"`

2. To create the policy, create a file named `role-policy.json` into your local with the following content:
   * Attention to `YOUR-BUCKET-HERE` which is the bucket you have uploaded your image to on the previous steps.
   * This policy allows to export the image to a separated bucket if you want to. You probably are not interested on exporting your image right now.
```
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect": "Allow",
         "Action": [
            "s3:GetBucketLocation",
            "s3:GetObject",
            "s3:ListBucket" 
         ],
         "Resource": [
            "arn:aws:s3:::YOUR-BUCKET-HERE",
            "arn:aws:s3:::YOUR-BUCKET-HERE/*"
         ]
      },
      {
         "Effect": "Allow",
         "Action": [
            "s3:GetBucketLocation",
            "s3:GetObject",
            "s3:ListBucket",
            "s3:PutObject",
            "s3:GetBucketAcl"
         ],
         "Resource": [
            "arn:aws:s3:::YOUR-BUCKET-HERE",
            "arn:aws:s3:::YOUR-BUCKET-HERE/*"
         ]
      },
      {
         "Effect": "Allow",
         "Action": [
            "ec2:ModifySnapshotAttribute",
            "ec2:CopySnapshot",
            "ec2:RegisterImage",
            "ec2:Describe*"
         ],
         "Resource": "*"
      }
   ]
}
```
Then run the following command:
`aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document "file://role-policy.json"`

### Create AMI
Now that we have a role with the required permission to import an image, let's import the image.
1. Create a file named `containers.json` to describe the containers with the following content:
   * Attention to `YOUR-BUCKET-HERE`. This is where your image was uploaded to.
   * Also, `YOUR-VM-KEY-HERE` is the key you used to upload the image.
```
[
  {
    "Description": "My Server OVA",
    "Format": "ova",
    "UserBucket": {
        "S3Bucket": "YOUR-BUCKET-HERE",
        "S3Key": "YOUR-VM-KEY-HERE"
    }
  }
]
```

2. Now run the following command:
`aws ec2 import-image --description "SOME DESCRIPTION TO YOUR AMI" --disk-containers "file://containers.json"`
   * This should output informations about the importing task containing the `ImportTaskId`. Use the `ImportTaskId` to check the status of the importing running the following command: `aws ec2 describe-import-image-tasks --import-task-ids YOUR-IMPORT-TASK-ID-HERE`

## Creating an EC2 instance using the imported image
You can do that using the console. I will write the steps to do that using the CLI later.


[split invocation]: https://www.gnu.org/software/coreutils/manual/html_node/split-invocation.html#split-invocation
[s3 multipart upload]: https://aws.amazon.com/blogs/aws/amazon-s3-multipart-upload/
[import-image]: https://docs.aws.amazon.com/vm-import/latest/userguide/vmimport-image-import.html#import-image-prereqs
