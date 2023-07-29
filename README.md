
Note that the “backend” code is commented because, as previously written, it assumes the existence of the s3 bucket and the dynamoDB table.



Create s3.tf
=============
We will create an s3 bucket called “any-name-is-fine” (you can call it as you want but remembering that the names of the s3 buckets in AWS are global, which means that it is not possible to use a name that has been used by someone else). In this case I decided to enable versioning so that every revision of the state file is stored, and is possible to roll back to an older version if something goes wrong. I decided to encrypt the contents of the bucket as the state file saves the infrastructure information and therefore also the sensitive data in plain-text. I also decided to enable the lock of objects in order to avoid deletion or overwriting.
resource "aws_kms_key" "mykey" {
  description             = "This key is used to encrypt bucket objects"
  deletion_window_in_days = 10
}

resource "aws_s3_bucket" "mybucket" {
  bucket = "mybucket-zar07172023"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "example" {
  bucket = aws_s3_bucket.mybucket.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.mykey.arn
      sse_algorithm     = "aws:kms"
    }
  }
}
dynamodb.tf
===========
We will create a table in dynamoDB called “any-name-db-state” (also in this case you can choose a different name, unlike the bucket, the name of the table is not global and therefore there can be multiple tables with the same name, as long as are not in the same region of the same account). The primary key to be used to lock the state in dynamoDB must be called LockID and must be a “string” type (S).
resource "aws_dynamodb_table" "dynamodb-terraform-state-lock" {
  name = "terraform-state-lock-dynamo"
  hash_key = "LockID"
  read_capacity = 20
  write_capacity = 20
 
  attribute {
    name = "LockID"
    type = "S"
  }
}
vpc.tf
======
Created these two files we can now create another file to deploy for example a very simple vpc. Let’s create a file called “vpc.tf” and past the following code:

/*# Backend must remain commented until the Bucket
 and the DynamoDB table are created. 
 After the creation you can uncomment it,
 run "terraform init" and then "terraform apply" */

 #terraform {
   #backend "s3" {             
     #bucket         = "mybucket-zar07172023"
     #key            = "terraform.tfstate"
     #region         = "us-east-1"
     #dynamodb_table = "terraform-state-lock-dynamo"
   #}
 #}

resource "aws_vpc" "vpc" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"
  tags = {
    Name = "vpc"
  }
}


Initializing the backend...
╷
│ Error: Error acquiring the state lock
│
│ Error message: 2 errors occurred:
│       * ResourceNotFoundException: Requested resource not found
│       * ResourceNotFoundException: Requested resource not found

RESOLUTION
===========
To resolve this issue, you should verify the following:

DynamoDB Table Name: Make sure that the DynamoDB table name 
specified in your Terraform backend configuration 
(dynamodb_table = "terraform-state-lock") matches the 
actual DynamoDB table you created for state locking. 
Ensure that the table name is correct and does not contain 
any typos or extra spaces.

CLEAN-UP
======

│ Error: deleting Amazon S3 (Simple Storage) Bucket (mybucket-zar07172023): BucketNotEmpty: The bucket you tried to delete is not empty
│       status code: 409, request id: 34J6YF4Q1VD4G5NR, host id: re7TJWUhA1zhyh7gVYdOR7bVOV/tBIb4CsvFMcsGAYe5FQAvRIG6sU8rUB14wW5ipTrRSia/fYI=       
│
│
╵
╷
│ Error: Error releasing the state lock
│
│ Error message: failed to retrieve lock info: ResourceNotFoundException: 
│ Requested resource not found
│
│ Terraform acquires a lock when accessing your state to prevent others   
│ running Terraform to potentially modify the state at the same time. An  
│ error occurred while releasing this lock. This could mean that the lock 
│ did or did not release properly. If the lock didn't release properly,   
│ Terraform may not be able to run future commands since it'll appear as  
│ if
│ the lock is held.
│
│ In this scenario, please call the "force-unlock" command to unlock the  
│ state manually. This is a very dangerous operation since if it is done  
│ erroneously it could result in two people modifying state at the same   
│ time.
│ Only call this command if you're certain that the unlock above failed   
│ and
│ that no one else is holding a lock.
============================================================
All resources ended up been deleted witrhout any specific intervention i an remember