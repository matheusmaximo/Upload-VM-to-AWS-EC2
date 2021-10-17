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
* Create a multipart file
* Upload each part
* Complete the upload
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

[split invocation]: https://www.gnu.org/software/coreutils/manual/html_node/split-invocation.html#split-invocation
[s3 multipart upload]: https://aws.amazon.com/blogs/aws/amazon-s3-multipart-upload/
