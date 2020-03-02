# plume

Fedora CoreOS and Fedora Cloud release utility

## Pre-flight

### AWS

Before you can create AMIs you need to ensure you have the following created:

* S3 bucket
* VM Import Service Role
* VM role-policy
* Optional: Allow VM Import Service Role to read KMS for disk encryption

#### Create S3 Bucket

Create a S3 bucket:

`aws s3 mb s3://my-cool-example-plume-bucket --region us-east-2`

Replace `my-cool-example-plume-bucket` with your unique S3 bucket name. Make sure to put in the correct region.

#### Create Import Service Role:

In order for plume to be able to create AMIs, it needs a role and role policy created. You can read more about it in detail [here](
https://docs.aws.amazon.com/vm-import/latest/userguide/vmimport-image-import.html#import-vm).

Save this to `vmimport-trust-policy.json`:

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

Use this file to create the new plume role:

`aws iam create-role --role-name plume --assume-role-policy-document "file://vmimport-trust-policy.json"`


Save this to `plume-role-policy.json` and replace `my-cool-example-plume-bucket` with the correct S3 bucket name:

```
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Action":[
            "s3:GetBucketLocation",
            "s3:GetObject",
            "s3:ListBucket"
         ],
         "Resource":[
            "arn:aws:s3:::my-cool-example-plume-bucket",
            "arn:aws:s3:::my-cool-example-plume-bucket/*"
         ]
      },
      {
         "Effect":"Allow",
         "Action":[
            "ec2:ModifySnapshotAttribute",
            "ec2:CopySnapshot",
            "ec2:RegisterImage",
            "ec2:Describe*"
         ],
         "Resource":"*"
      }
   ]
}
```

Assign the plume role policy to the `vmimport` role:

`aws iam put-role-policy --role-name vmimport --policy-name plume --policy-document "file://plume-role-policy.json"`

Optional: Allow the plume role to use KMS to read encrypted S3 buckets.

If you get this error:

> The service role <vmimport> does not exist or does not have sufficient permissions for the service to continue

You might have S3 SSE enabled and you need to add this policy to the `plume` role.

Save this to: `plume-s3-sse-role-policy.json`:

```
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Action":[
            "kms:Decrypt"
         ],
         "Resource":"*"
      }
   ]
}

```

`aws iam put-role-policy --role-name plume --policy-name plume-s3-sse --policy-document "file://plume-s3-sse-role-policy.json"`


## Testing

### Build a release image with the SDK

```sh
export COREOS_BUILD_ID=$(date +%Y-%m-%d-%H%M)
KEYID="<keyid>"
gpg2 --armor --export "$KEYID" > ~/keyfile
./build_packages
./build_image --upload --sign="$KEYID" prod
for format in ami_vmdk azure gce; do
    ./image_to_vm.sh --format=$format --upload --sign="$KEYID"
done
```

### Perform the "release"

```sh
bin/plume pre-release -C user --verify-key ~/keyfile -V $version-$COREOS_BUILD_ID
bin/plume release -C user -V <version>-$COREOS_BUILD_ID
```

### Fedora Cloud Releases

There are two sub-commands to do a Fedora Cloud Release: `pre-release` and `release`. When using `pre-release` plume will download the compose-id from the specified channel, extract the contents, and upload it to S3. From there it will use the import VM feature within AWS to create a private AMI from S3 and it will make it available across all regions. When using `release` it will make these AMIs public.

Here is an example of doing a Fedora Cloud pre-release with plume:

```
./bin/plume pre-release                   \
  --distro fedora                         \
  --channel cloud                         \
  --version 30                            \
  --timestamp 20190819                    \
  --respin 0                              \
  --board x86_64                          \
  --compose-id Fedora-Cloud-30-20190819.0 \
  --image-type Cloud-Base                 \
  --debug
```

Here is an example of doing a Fedora Cloud release with plume:

```
./bin/plume release                       \
  --distro fedora                         \
  --channel cloud                         \
  --version 30                            \
  --timestamp 20190819                    \
  --respin 0                              \
  --board x86_64                          \
  --compose-id Fedora-Cloud-30-20190819.0 \
  --image-type Cloud-Base                 \
  --debug
```

### Clean up

Delete:

- Stuff uploaded into `gs://users.developer.core-os.net/$USER`
- GCE image in `coreos-gce-testing`
- AWS AMIs and snapshots in `us-west-1`, `us-west-2`, and `us-east-2`