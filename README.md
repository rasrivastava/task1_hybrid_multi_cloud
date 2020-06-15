/* 
Task 1 : Have to create/launch Application using Terraform

1. Create the key and security group which allow the port 80.
2. Launch EC2 instance.
3. In this Ec2 instance use the key and security group which we have created in step 1.
4. Launch one Volume (EBS) and mount that volume into /var/www/html
5. Developer have uploded the code into github repo also the repo has some images.
6. Copy the github repo code into /var/www/html
7. Create S3 bucket, and copy/deploy the images from github repo into the s3 bucket and change the permission to public readable.
8 Create a Cloudfront using s3 bucket(which contains images) and use the Cloudfront URL to  update in code in /var/www/html
*/

```
$ cat variables.tf 
variable profile_name {
    default = "rasrivasprofile"
}

variable ssh_key_name {
    default = "EC2KeyPair"
}

variable firewall_name {
    default = "MySecurityGroup"
}


variable ami_id {	
    default = "ami-0447a12f28fddb066"
}


variable instance_name {
    default = "MyWebServer"
}


variable bucket_name {
	default = "rasrivas-bucket-1234"
}


variable object_name {
    default = "image.jpg"
}
```

```
$ cat install_pkg_instance.sh 
#!/bin/bash

sudo yum install httpd php git -y
sudo systemctl restart httpd
sudo systemctl enable httpd
```

```
$ cat ebs_vol_operation.sh 
#!/bin/bash

sudo mkfs.ext4  /dev/xvdh
sudo mount  /dev/xvdh  /var/www/html
sudo rm -rf /var/www/html/*
sudo git clone https://github.com/rasrivastava/tf_test.git /var/www/html/
```

