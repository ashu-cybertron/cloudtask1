# cloudtask1

Terraform is a tool which gives us a standard platform to manage all the clouds like aws, azure . First thing first , why we need terraform in this time small and big industries , startups need cloud service providers to manage the services for them because they don’t want to involve in managing system architecture. So, they can think about policies and profit.
And there are many cloud service providers so different provider has different packages for different services so the problem was , How to learn all the terminology and language of all cloud providers . Here Terraform comes in play . It provides a single language HCL to manage most of the cloud services.
And, Now come to our project given as a task by World Record Holder Vimal Daga sir in our Hybrid Multi Cloud Internship and Training.
What is to be done in this let’s see —

Task : Have to create/launch Application using Terraform
1. Create the key and security group which allow the port 80.
2. Launch EC2 instance.
3. In this Ec2 instance use the key and security group which we have created in step 1.
4. Launch one Volume (EBS) and mount that volume into /var/www/html
5. Developer have uploded the code into github repo also the repo has some images.
6. Copy the github repo code into /var/www/html
7. Create S3 bucket, and copy/deploy the images from github repo into the s3 bucket and change the permission to public readable.
8 Create a Cloudfront using s3 bucket(which contains images) and use the Cloudfront URL to update in code in /var/www/html
Optional
1) Those who are familiar with jenkins or are in devops AL have to integrate jenkins in this task wherever you feel can be integrated
2) create snapshot of ebs ,Above task should be done using terraform
And , What i have done is :
First of all, configured my AWS profile in local system using Command Prompt. Filled details then Enter.

aws configure --profile ashu
AWS Access Key ID [****************5KMO]:
AWS Secret Access Key [****************VpMS]:
Default region name [ap-south-1]:
Default output format [None]:

here i already entered the access and secret keys so it is hidden.
Second work is to be done is :
Launch an ec2 instance using Terraform. I have used a Amazon AMI 2. in this installed and configured webserver in instance using Remote Exec Provisioner . i used the my key created earlier . The code for this is -

provider  "aws" {
        region   = "ap-south-1"
        profile  = "ashu"
      }

      resource "aws_instance" "myin" {
        ami             =  "ami-0447a12f28fddb066"
        instance_type   =  "t2.micro"
        key_name        =  "ashu12345"
        security_groups =  [ "launch-wizard-2" ]

       connection {
          type     = "ssh"
          user     = "ec2-user"
          private_key = file("C:/Users/Lenovo/Downloads/ashu12345.pem")
          host     = aws_instance.myin.public_ip
        }

        provisioner "remote-exec" {
          inline = [
            "sudo yum install httpd  php git -y",
            "sudo systemctl restart httpd",
            "sudo systemctl enable httpd",
            "sudo setenforce 0"
          ]
        }

        tags = {
          Name = "ashuos"
        }
      }
      
Third work is : Create an EBS volume. we need to launch our EBS volume in the same zone what we selected in above steps because they can't be connected without it . For this, I have retrieved the availability zone of the instance & used it here.

resource "aws_ebs_volume" "ashuvol" {
        availability_zone  =  aws_instance.myin.availability_zone
        size               =  1

        tags = {
          Name = "ashuebs"
        }
      }
      
Fourth Work Is : Attach EBS volume to the instance.

resource "aws_volume_attachment"  "ebs_att" {
        device_name  = "/dev/sdd"
        volume_id    = "${aws_ebs_volume.ashuvol.id}"
        instance_id  = "${aws_instance.myin.id}"
        force_detach =  true
      }
      
I have retrieved the public ip and stored it in a file in my system.

resource "null_resource" "public_ip"  {
        provisioner "local-exec" {
            command = "echo  ${aws_instance.myin.public_ip} > public_ip.txt"
          }
      }
      
Work 5 is : mount EBS volume to the folder /var/www/html.

resource "null_resource" "mount"  {

    depends_on = [
        aws_volume_attachment.ebsatt,
      ]


      connection {
        type     = "ssh"
        user     = "ec2-user"
        private_key = file("C:/Users/Lenovo/Downloads/ashu12345.pem")
        host     = aws_instance.myin.public_ip
      }

    provisioner "remote-exec" {
        inline = [
          "sudo mkfs.ext4  /dev/xvdd",
          "sudo mount  /dev/xvdd  /var/www/html",
          "sudo rm -rf /var/www/html/*",
          "sudo git clone https://github.com/ashu-cybertron/cloudtask1 /var/www/html/"
        ]
      }
    }
    
I have downloaded all the code ,images from Github in my local system.

resource "null_resource" "git_copy"  {
      provisioner "local-exec" {
        command = "git clone https://github.com/ashu-cybertron/cloudtask1 C:/Users/Lenovo/Pictures/" 
        }
    }
Work 6 : create an S3 bucket on AWS.
resource "aws_s3_bucket" "ashubkt" {
        bucket = "ashu123"
        acl    = "private"

        tags = {
          Name        = "ashu1234"
        }
      }
       locals {
          s3_origin_id = "myS3Origin"
        }
Work 7 : S3 bucket created, upload image that downloaded from Github in local system. I uploaded one . You can upload more.
resource "aws_s3_bucket_object" "object" {
          bucket = "${aws_s3_bucket.ashubkt.id}"
          key    = "test_pic"
          source = "C:/Users/Lenovo/Pictures/img1.jpg"
          acl    = "public-read"
        }
        
Work 8 : create a CloudFront and connect it to S3 bucket. The CloudFront is needed for fast delievery of content from the edge locations across the world.

resource "aws_cloudfront_distribution" "ashufnt" {
         origin {
               domain_name = "${aws_s3_bucket.ashubkt.bucket_regional_domain_name}"
               origin_id   = "${local.s3_origin_id}"

       custom_origin_config {

               http_port = 80
               https_port = 80
               origin_protocol_policy = "match-viewer"
               origin_ssl_protocols = ["TLSv1", "TLSv1.1", "TLSv1.2"] 
              }
            }
               enabled = true

       default_cache_behavior {

               allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
               cached_methods   = ["GET", "HEAD"]
               target_origin_id = "${local.s3_origin_id}"

       forwarded_values {

             query_string = false

       cookies {
                forward = "none"
               }
          }

                viewer_protocol_policy = "allow-all"
                min_ttl                = 0
                default_ttl            = 3600
                max_ttl                = 86400

      }
        restrictions {
               geo_restriction {
                 restriction_type = "none"
                }
           }
       viewer_certificate {
             cloudfront_default_certificate = true
             }
      }
      
Now, go to /var/www/html and update the link of the images with the link of CloudFront.
Work 9 : Written a terraform code retrieve the public ip instance and open it in chrome. This will open the page of site present in /var/www/html.

resource "null_resource" "local_exec"  {


        depends_on = [
            null_resource.mount,
          ]

          provisioner "local-exec" {
              command = "start chrome  ${aws_instance.myin.public_ip}"
                 }
        }
        
Finally its done , it will open the home page and you can see the images what you uploaded in earlier steps.
Because of shortage of time i can not make it with jenkins i will do it alter after my some other college stuffs.
Thanks to VImal daga sir taught from very basic so i can create this project…
I have some more projects to do wait for the next one the journey is just started…
