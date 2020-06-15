Task 1 : Have to create/launch Application using Terraform
===========================================================

1. Create the key and security group which allow the port 80.
2. Launch EC2 instance.
3. In this Ec2 instance use the key and security group which we have created in step 1.
4. Launch one Volume (EBS) and mount that volume into /var/www/html
5. Developer have uploded the code into github repo also the repo has some images.
6. Copy the github repo code into /var/www/html
7. Create S3 bucket, and copy/deploy the images from github repo into the s3 bucket and change the permission to public readable.
8 Create a Cloudfront using s3 bucket(which contains images) and use the Cloudfront URL to  update in code in /var/www/html

- Variablea used in the main configuration file, user can edit as per there choice:

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

- Script which will install the required packages for apache web server, PHP and git on the instance

```
$ cat install_pkg_instance.sh 
#!/bin/bash

sudo yum install httpd php git -y
sudo systemctl restart httpd
sudo systemctl enable httpd
```

- Script which will format and mount the persistent volume and clone the web pages to the apache root directroy on the instance

```
$ cat ebs_vol_operation.sh 
#!/bin/bash

sudo mkfs.ext4  /dev/xvdh
sudo mount  /dev/xvdh  /var/www/html
sudo rm -rf /var/www/html/*
sudo git clone https://github.com/rasrivastava/tf_test.git /var/www/html/
```

- Lets start the main configuration


- Reference: https://www.terraform.io/docs/providers/aws/index.html
  - The Amazon Web Services (AWS) provider is used to interact with the many resources supported by AWS. The provider needs to be configured with the proper credentials before it can be used.

	```
	provider "aws" {
	    region = "ap-south-1"
	    profile = "${var.profile_name}"
	}
	```

- Reference: https://www.terraform.io/docs/providers/tls/r/private_key.html 
- Resource: aws_key_pair
  - Generates a secure private key and encodes it as PEM. This resource is primarily intended for easily bootstrapping throwaway development environments.
    - algorithm - (Required) The name of the algorithm to use for the key. Currently-supported values are "RSA" and "ECDSA".
    - rsa_bits - (Optional) When algorithm is "RSA", the size of the generated RSA key in bits. Defaults to 2048.
    - ecdsa_curve - (Optional) When algorithm is "ECDSA", the name of the elliptic curve to use. May be any one of "P224", "P256", "P384" or "P521", with "P224" as the default.

	```
	resource "tls_private_key" "private_key_pair" {
	    algorithm = "RSA"
	}
	```


- Reference: https://www.terraform.io/docs/providers/local/r/file.html
- Resource: local_file
  - Generates a local file with the given content.
    - content - (Optional) The content of file to create. Conflicts with sensitive_content and content_base64.
    - filename - (Required) The path of the file to create.
    - file_permission - (Optional) The permission to set for the created file. Expects an a string. The default value is "0777".

	```
	resource "local_file" "local_private_key" {
	    depends_on = [
			tls_private_key.private_key_pair
		]
	    content = tls_private_key.private_key_pair.private_key_pem
	    filename = 	"${var.ssh_key_name}.pem"
	    file_permission = "0400"
	}
	```


- Refernece: https://www.terraform.io/docs/providers/aws/r/key_pair.html
- Resource: aws_key_pair
  - Currently this resource requires an existing user-supplied key pair. This key pair public key will be registered with AWS to allow logging-in to EC2 instances.

    - key_name   - (Optional) The name for the key pair.
    - key_name_prefix - (Optional) Creates a unique name beginning with the specified prefix. Conflicts with key_name.
    - public_key - (Required) The public key material.
    - tags - (Optional) Key-value map of resource tags

	```
	resource "aws_key_pair" "key_pair" {
	    depends_on = [
			local_file.local_private_key
		]
	    key_name = var.ssh_key_name
	    public_key = tls_private_key.private_key_pair.public_key_openssh
	}
	```


- Refernece: https://www.terraform.io/docs/providers/aws/r/security_group.html
- Resource: aws_security_group
  - name - (Optional, Forces new resource) The name of the security group. If omitted, Terraform will assign a random, unique name 
  - description - (Optional, Forces new resource) The security group description.
  - ingress - (Optional) Can be specified multiple times for each ingress rule. Each ingress block supports fields documented below.
  - egress - (Optional, VPC only) Can be specified multiple times for each egress rule. Each egress block supports fields documented below.
    - The ingress/egress block supports:
      - from_port - (Required) The start port (or ICMP type number if protocol is "icmp" or "icmpv6")
      - to_port - (Required) The end range port (or ICMP code if protocol is "icmp").
      - protocol - (Required) The protocol. If you select a protocol of "-1" (semantically equivalent to "all", which is not a valid value here), you must specify a "from_port" and "to_port" equal to 0.
      - cidr_blocks - (Optional) List of CIDR blocks.

		```
		resource "aws_security_group" "firewall" {
		    depends_on = [
			aws_key_pair.key_pair
		    ]
		    name = var.firewall_name
		    description = "Allow HTTP and SSH inbound traffic"
		    ingress	{
			from_port = 80
			to_port = 80
			protocol = "tcp"
			cidr_blocks = ["0.0.0.0/0"]
		    }
		    ingress {
			from_port = 22
			to_port = 22
			protocol = "tcp"
			cidr_blocks = ["0.0.0.0/0"]
		    }
		    egress {
			from_port = 0
			to_port = 0
			protocol = "-1"
			cidr_blocks = ["0.0.0.0/0"]
		    }
		}
		```
