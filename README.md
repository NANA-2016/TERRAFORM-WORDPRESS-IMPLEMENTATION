# TERRAFORM-WORDPRESS-IMPLEMENTATION 

## ACCHIEVEMENTS

### NETWORKING

Created a VPC with public and private subnet

Public subnet contains ALB,NAT GATEWAY with inbound rules and outbound to private subnets

<img width="1881" height="797" alt="image" src="https://github.com/user-attachments/assets/202898ee-da32-400a-a01c-18f4dc693c23" />

<img width="1892" height="791" alt="image" src="https://github.com/user-attachments/assets/b148bd4c-bd08-4a06-9922-6c713760ca02" />

Private subnet contains EC2 WEBTIER,RDS EFS . It can only be accessed through public subnet. with out bound via NAT GATEWAY only

<img width="1780" height="482" alt="image" src="https://github.com/user-attachments/assets/222301ae-4033-4e9b-abe1-57aba102730d" />

Below are the instances created:

<img width="1642" height="480" alt="image" src="https://github.com/user-attachments/assets/6c008d22-d617-4dbe-9746-ba345bc8a373" />

## AWS MySQL,RDS setup

Private subnets created which are not publicly available with subgroups restricting to EC2 subgroups only

## EFS setup

 the variables used to set it up are and was set up to allow access upload across instances

 <img width="1891" height="617" alt="image" src="https://github.com/user-attachments/assets/30585a53-f0f5-4ae5-a74d-7c1c525f0622" />


 ##ALB(Application Load Balancer)

 It is the only one that is exposed to the internet for security purposes as it fronts the enviroment.

 <img width="1871" height="457" alt="image" src="https://github.com/user-attachments/assets/6eab759f-16af-4fa9-b60b-ebaab4cc8d1b" />


 ## Autoscaling Group

 It ensure the whole system is secure and alsp reduces the risk of overload

 <img width="1883" height="362" alt="image" src="https://github.com/user-attachments/assets/4227dc3b-3fcd-43c2-bc93-be058baf3f6a" />


 The commands used all throught to allow setting up are 

 Terraform Init -To help initialize terraform

 Terraform Validate- To help validate the code 

 terraform plan -out tfplan -var 'db_password=CHANGE_ME_Strong!Passw0rd'

 teraform apply tfplan
