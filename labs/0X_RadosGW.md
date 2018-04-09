# RadosGW

## S3 API

In this section we will configure and user RadosGW's S3 compatible API

## Create a user for S3 API

  * Create a test user

        radosgw-admin user create --uid s3test --display-name="Test user for S3 API"

  * This will create a user and it's S3 access_key and secret_key

## Use S3 API

These steps can be done from any host with access to the RadosGW node

  * Install libs3-tools 

        zypper in libs3-tools

  * Setup S3_AUTH environment variables (with the user credentials)

        export S3_ACCESS_KEY_ID=K9HE6BFD9BYCCL9NRHF8
        export S3_SECRET_ACCESS_KEY=QmQJAhmT33Ch1kE8FnIkoDMWtBvjQQhN8lBfbpOW
        export S3_HOSTNAME=192.168.122.21

  * Since RadosGW is using HTTP and s3 CLI uses HTTPS by default, we need to force HTTP adding the "-u" switch to all "s3" commands

  * List buckets

        s3 -u list

  * Create a test bucket

        s3 -u create test-bucket

  * Put an object

        echo "Test object" | s3 -u put test-bucket/test

  * Check the object as been created

        s3 -u list test-bucket

  * Get the object

        s3 -u get test-bucket/test

  * Delete de object

        s3 -u delete test-bucket/test

