# Continuous Integration / Deployment 

You know what the say ... If you are doing Continuous Deployment you must be doing Continuous Integration 

Deploy an EC2 Instance from Public AMI , AMI contains:

1. Ubuntu OS
2. Jenkins Server
3. Maven
4. JDK
5. AWS CLI Tools

See AMIID.md for the AMI ID.

The purpose is to demonstrate a CI/CD Simple pipeline, the flow:

```
Simple "Hello World!" Java APP ---> Remote Repo on GitHub
											|
											|
										   \|/
										 Jenkins Server Monitoring Repo for Changes --> Maven Builds Java APP --> Post Successful Build Hook Triggered
										              																		|
																															|
																														   \|/
																											Jar Artifact copied to s3 bucket			   
																											RunInstance API , Start new Instance
																											                |
																														   \|/			    
										                                                                       User-Data Pulls Jar Artifact
																											   And deploys Code
																											                |
																														   \|/
																											SNS Notification Triggered with exit code
```
Installation Details:

1. Hopefully you have a github account , if not create one ,  Create a new repo , The Jenkins server will use SSH Keys to connect to the repo
   So you will need to import your public key to the github config (I used new ssh keypair only for this purpose!)
2. Create an S3 Bucket , maybe YourName-jenkins 
3. Jenkins Instance IAM Role: Create an IAM Role with the below policies and attach the it to the CI-Server Instance created from the AMI
```
   Pass Role Permission Policy - We use the this policy to allow the CI-Server to execute RunInstance and pass IAM role to the newly created Instance

  {
     "Version": "2012-10-17",
     "Statement": [{
        "Effect":"Allow",
        "Action":["ec2:*"],
        "Resource":"*"
      },
      {
        "Effect":"Allow",
        "Action":"iam:PassRole",
        "Resource":"arn:aws:iam::xxxxAWS-Account-IDxxx:role/NameOFthe2ndRoleThatWillBeAttachedToTheLaunchedEC2Instance"
      }]
  }
  
  S3 Bucket Permission Policy - CI Server Needs to Put Jar Artifact Into an S3 Bucket
  
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "Stmt1403533488000",
        "Effect": "Allow",
        "Action": [
          "s3:ListBucket"
        ],
        "Resource": [
          "arn:aws:s3:::youname-jenkins"
        ]
      },
      {
        "Sid": "Stmt1403533560000",
        "Effect": "Allow",
        "Action": [
          "s3:Get*",
          "s3:Put*"
        ],
        "Resource": [
          "arn:aws:s3:::yourname-jenkins/*"
        ]
      }
    ]
  }
```
4. Application EC2 Instance IAM Role: Create an IAM Role with a policy that will allow the EC2 Instance to download the Jar Artifcat from S3
```
    S3 Read Only Bucket Policy
	
	{
	  "Version": "2012-10-17",
	  "Statement": [
	    {
	      "Sid": "Stmt1403534521000",
	      "Effect": "Allow",
	      "Action": [
	        "s3:ListBucket"
	      ],
	      "Resource": [
	        "arn:aws:s3:::YourName-jenkins"
	      ]
	    },
	    {
	      "Sid": "Stmt1403534565000",
	      "Effect": "Allow",
	      "Action": [
	        "s3:Get*"
	      ],
	      "Resource": [
	        "arn:aws:s3:::YourName-jenkins/*"
	      ]
	    }
	  ]
	}
	
	SNS Publish messsage
	
	{
	  "Version": "2012-10-17",
	  "Statement": [
	    {
	      "Sid": "Stmt1403543679000",
	      "Effect": "Allow",
	      "Action": [
	        "sns:Get*",
	        "sns:ListTopics",
	        "sns:Publish"
	      ],
	      "Resource": [
	        "arn:aws:sns:eu-west-1:xxxxAWS-Account-IDxxxX:YourTopic"
	      ]
	    }
	  ]
	}
```
5. Deploy the AMI , Do not forget to attach the IAM role created in step 1
6. Login using the ubuntu user and setup the repo and ssh keys : As I mentioned I used a separated SSH key pair for this demo for security reasons
   a. Create the github access: copy your private ssh key to /home/ubuntu/.ssh , and to /var/lib/jenkins/.ssh/
      using the ubuntu user goto /home/ubuntu/src/testme and initialize the repo, commit the first version and create the remote pointing to your repo on github
	  See github documentation for more info , Test the Git but commiting a change and pushing to remote
7. Edit the EC2 Instance Script: Since Jenkins will start an EC2 instance upon a successful build and will send an SNS message you will need to
   Provide several parameters: as the ubuntu user edit /opt/scripts/user-data.sh , change the s3 bucket name to match the one you created 
   On the aws SNS command , input your AWS Account number and the ARN of the topic , make sure you are subscribed with an email subscriber
8. Browse to the jenkins UI http://ip:8080 , on the landing page you should see the AWS CI Demo build job  , hoover and open the command menu enter configuration
   On the source code management-->Repository URL--> input your repo like this:  git@github.com:GITUSER/myrepo.git 
   Then make sure that on the "Credentials" Option the Jenkins user is chosen
9. On the POST Steps --> Execute Shell this is where the magic happens and after a successful build this code snippet will fire note the below command
   sudo /opt/scripts/newInstance.sh -a ami-892fe1fe -s sg-0964a56c -k MasterNew -i t2.micro -n subnet-c7c3cbb3 -m "Arn=arn:aws:iam::xxxxAWS-Account-IDxxxX:instance-profile/RoleNameFromStep3"
   
   Make sure you replace -k with your own keypair and -n with a valid subnet inside your region the -m will be the role instance profile name you created 
   In step 3
10. On the POST Steps --> Execute Shell , update your bucket name
   aws --region eu-west-1 s3 cp target/testme-1.0-SNAPSHOT.jar s3://YourName-jenkins/

Well at this point every code change to App.example Class which will be pushed to git remote repo should trigger jenkins job to build the job and start 
And EC2 + Deploy the Code

For Questions feel free to reach out to me :  kobibito@amazon.lu 
