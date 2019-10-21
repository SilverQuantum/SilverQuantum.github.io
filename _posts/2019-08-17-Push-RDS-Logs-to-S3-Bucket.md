---
layout: post
title: Pushing AWS RDS logs into S3 bucket
tags: [AWS, RDS database, S3, log management]
bigimg: 
---

# Abstract
Using AWS RDS has added ease of database infrastructure setup and administration. As an Ops-engineer you may need to manage the database log files and archive them to storage. In this post I will discuss backing up RDS logfile to S3 bucket. This post is focused on general ideas to make the backup process succeed.

---

First, we need to create an IAM group in AWS with RDS and S3 full access (I am using AmazonRDSFullAccess and AmazonS3FullAccess policies - although its not recommended to use S3 full access policy but I am trying to implement a basic general idea of backup process). 

Next, create IAM user with AWS access key. We will be using this access key in aws CLI client (instead of using access credential in EC2 use IAM role while creating EC2 instance for better security). 

Then, install aws cli using pip and configure aws cli using '**aws configure**' command. 

Time to use backup script for testing:

> 1  #!/bin/bash
> 2  PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/home/ubuntu
> 3 
> 4  INSTANCE='rds-db-identifier'
> 5  BUCKET='backup-db-log-bucket'
> 6  
> 7  mkdir -p ${INSTANCE} && cd ${INSTANCE}
> 8  for i in `/home/ubuntu/.local/bin/aws rds describe-db-log-files --db-instance-identifier ${INSTANCE} --output text | awk '{print $3}' | sed '$d' | tail -n 10` ; do
> 9  	FILE=`basename ${i}`
> 10	ARCHIVE=${FILE}.tar.gz
> 11	if [ ! -e ${ARCHIVE} ]; then
> 12		echo "Downloading ${i} ........."
> 13		`which aws` rds download-db-log-file-portion --db-instance-identifier ${INSTANCE} --log-file-name ${i} --starting-token 0 --output text > ${FILE}
> 14		tar -cvzf ${ARCHIVE} ${FILE}
> 15		echo "Pushing to S3 bucket ........"
> 16		`which aws` s3 mv ${ARCHIVE} s3://${BUCKET}/
> 17		rm ${FILE}
> 18	fi
> 19done

Write this script to a specific location with the name *backup*. Next provide execute permission to the file using _**chmod u+x ./path-to-script**_ for making the script to work. Make sure that you have provided authentic bucket name and db instance name. Now execute the script and you will see that rds logs from specific instance are getting downloaded in the local then getting archived and uploaded to S3 bucket with an end of file removal. 

You may use cronjob to automate the backup process after a fixed interval as following:

> 1| $ crontab -e
> 2| $ */45 * * * * /bin/bash /home/ubuntu/.crontasks/backup >/dev/null 2>&1 


