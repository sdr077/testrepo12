# testrepo12
# AppMonitor
About this 

This is Demo web app hosting and taking logs file whenever there is an issue in the servers. Those files copy into s3 bucket. If there is web server not running within given time period automatically it will start and requested down time it will stop and create the log file. If there is an issue while copying s3 bucket system will send an email to support team

Server configuration Installation
1.	<a href="http://httpd.apache.org/docs/2.4/install.html">Apache installing</a>
2.	<a href="https://www.howtoforge.com/tutorial/configure-postfix-to-use-gmail-as-a-mail-relay/">Mail Server configuration </a>
3.	<a href="https://www.digitalocean.com/community/tutorials/how-to-install-mysql-on-ubuntu-20-04">MySqlServer insallation</a>
<br> 

###Server Scripts####

##Web Server##
```
Start-server.sh
#!/bin/bash
date=$(date)
today="$( date +"%Y%m%d" )"
filename="webserver-$today.log"
serverdns="<server dns>"

sudo systemctl start httpd

status="200"
statusmessage="SUCCESS::Server Started..."
content=$(curl $serverdns)
#### Writing to log file###
if [ ! -e /home/ec2-user/logs/$filename ]; then
	echo "$date:::STATUS:$status:::$statusmessage:::{Message : $content}" > /home/ec2-user/logs/$filename
else
        echo "$date:::STATUS:$status:::$statusmessage:::{Message : $content}" >> /home/ec2-user/logs/$filename
fi
```

```
stop-server.sh

#!/bin/bash
date=$(date)
today="$( date +"%Y%m%d" )"
filename="webserver-$today.log"
serverdns="<server dns>"

sudo systemctl stop httpd

status="404"
statusmessage="SUCCESS::Server Stopped..."
content="NULL"
#### Writing to log file###
if [ ! -e /home/ec2-user/logs/$filename ]; then
	echo "$date:::STATUS:$status:::$statusmessage:::{Message : $content}" > /home/ec2-user/logs/$filename
else
        echo "$date:::STATUS:$status:::$statusmessage:::{Message : $content}" >> /home/ec2-user/logs/$filename
fi

```

###External Server###


External Server (Jump Server)

External server is consists with 4 bash files
1.	<a href="https://github.com/sdr077/AppMonitor/blob/main/bash/cronlogcopy.sh">cronlogcopy.sh</a>
2.	<a href="https://github.com/sdr077/AppMonitor/blob/main/bash/cronmonitor.sh">cronmonitor.sh</a>
3.	<a href="https://github.com/sdr077/AppMonitor/blob/main/bash/s3-copy.sh">s3-copy.sh</a>
4.	<a href="https://github.com/sdr077/AppMonitor/blob/main/bash/server.sh">server.sh</a>
<br>
Out of these 4 scripts cronmonitor.sh and cronlogcopy.sh are the cronjob scripts which are running on a schedule.  These two scripts are added to the crontab.
cronmonitor.sh remotely executes the server.sh file where it checks the web server status every 5mintues. 


```
cronlogcopy.sh

#!/bin/bash

ssh <server dns> -i access.key 'bash -s' < s3-copy.sh

```

```

cron-monitor.sh

#!/bin/bash

ssh <server dns> -i access.key 'bash -s' < server.sh
```

```
s3-copy.sh

#!/bin/bash
content_path="/var/www/html/index.html"
today="$( date +"%Y%m%d" )"
log_path="/home/ec2-user/logs/webserver-$today.log"

##Start th zipping the files
ls $log_path
zip  -j "/home/ec2-user/logs/$today-log+content-backup".zip $log_path $content_path

##Copy to s3 bucket
aws s3 cp "/home/ec2-user/logs/$today-log+content-backup.zip" s3://webserver-logs-monitor/webserver-logs/

##Checking the file is there in the s3 and delete the compressed file
sleep 20
exists=$(aws s3 ls s3://webserver-logs-monitor/webserver-logs/$today-log+content-backup.zip)
if [ -z "$exists" ]; then
	echo "Log archive did not copy to S3 please check. Email sent!"	 
       	echo "Log archive did not copy to S3 please check"  | mail -s "AWS S3 log alert : `date`" mail.com
  else
	  echo "Removing old log archive" 
	  rm -rf /home/ec2-user/logs/$today-log+content-backup.zip
fi

```

```
server.sh

#!/bin/bash
date=$(date)
today="$( date +"%Y%m%d" )"
filename="webserver-$today.log"
serverdns="<serverdns>"
hour="$(date +"%k")"
minute="$(date +"%M")"
DB_NAME="results"
TABLE="timestamp_tbl"

healthmonitor(){	
status=$(curl -s -w '%{http_code}' -o /dev/null $serverdns)

if [ $status == "200" ]; then
	echo "Success : WebServer is running"
	echo "Checking the webpage content..."
	echo "====================================================================="
	curl -i $serverdns
	content=$(curl $serverdns)
	sleep 3
	echo "====================================================================="
	statusmessage="SUCCESS::Server is running"
else

	echo "Error : WebServer is not running!!"
	echo "Starting the WebServer..."
	sudo systemctl start httpd
	sleep 5
	status=$(curl -s -w '%{http_code}' -o /dev/null $serverdns)
	if [ $status == "200" ]; then
		echo "Success : WebServer is running"
		echo "Checking the webpage content..."
		echo "====================================================================="
		curl -i $serverdns
		sleep 3
		echo "====================================================================="
		echo "Server got rebooted"  | mail -s "SERVER ALERTS : `date`" mail.com
		statusmessage="WARNING::Server is restarted"
		content=$(curl $serverdns)
		status="200"
	else 
		echo "Error : WebServer is not started. Please check with the support team"
		statusmessage="ERROR::Server ran into issue"
		echo "ERROR::Server ran into issues, httpd service can't be started. Please log into the server and check"  | mail -s "SERVER ALERTS : `date`" mail.com
		status="404"
		content="not available"
	fi
fi
}

## Checking whether the time is between 09:00AM and 05:00PM ##
if [ $hour -eq 11 ]; then
	if [ $minute -ge 30 ]; then
		echo "Health Check Script only run during 9:00AM to 5:00PM IST"
		exit 1 
	else
		healthmonitor
	fi
elif [ $hour -gt 11 ]; then
	echo "Health Check Script only run during 9:00AM to 5:00PM IST"
        exit 1
elif [ $hour -eq 3 ]; then
	if [ $minute -lt 30 ]; then
       		echo "Health Check Script only run during 9:00AM to 5:00PM IST"
                exit 1
	else
		healthmonitor
	fi
elif [ $hour -lt 3 ]; then
        echo "Health Check Script only run during 9:00AM to 5:00PM IST"
        exit 1	
else
	healthmonitor
fi

#### Writing to log file###
if [ ! -e /home/ec2-user/logs/$filename ]; then
   echo "$date:::STATUS:$status:::$statusmessage:::{Message : $content}" > /home/ec2-user/logs/$filename
 else
   echo "$date:::STATUS:$status:::$statusmessage:::{Message : $content}" >> /home/ec2-user/logs/$filename
fi

#### Writing to MySQL database###
mysql --login-path=client $DB_NAME -e "INSERT INTO $TABLE (\`status_code\`, \`status_message\`, \`content\`, \`timestamp_ts\`) VALUES ('$status','$statusmessage','$content','$date')"


```

######## Server provisioning using Terraform #############


```
main.tf

# Let AZs be shuffled when more than 1 AZ is needed.
resource "random_shuffle" "az" {
  input = var.availability_zones
  result_count = length(var.availability_zones)
}


#create new vpc with 10.0.0.0/16  cidr block

resource "aws_vpc" "main" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"

  tags = {
    Name = "Demo VPC"
  }
}

# DB Subnet.
resource "aws_subnet" "db_subnet" {  
  count = 2
  cidr_block = var.subnet_cidr_blocks.db[count.index] 
  vpc_id = aws_vpc.main.id 
  availability_zone = random_shuffle.az.result[count.index % length( random_shuffle.az.result)]
   tags = {
    Name = "DB-Subnet"
  }
  
}

# App Subnet  w/ NAT Gateway.
resource "aws_subnet" "app_subnet" {  
  count = 2
  cidr_block = var.subnet_cidr_blocks.app[count.index] 
  vpc_id = aws_vpc.main.id  
  availability_zone = random_shuffle.az.result[count.index % length( random_shuffle.az.result)]
  tags = {
    Name = "App-Subnet"
  }  
}

# NAT Gatway w/ EIP.
resource "aws_eip" "nat_eip" {
  vpc = true
  tags = {
    Name = "nat-eip"
  } 
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id = aws_subnet.app_subnet[0].id
  
}

# IGW w
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "Internate-Gateway"
  } 
}

resource "aws_route_table" "igw_private_route_table" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "private-route-table"
  } 
}

# Route table for NAT.
resource "aws_route_table" "nat_private_route_table" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "public-route-table"
  } 
}

resource "aws_route" "route_to_internet" {
  route_table_id = aws_route_table.igw_private_route_table.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id = aws_internet_gateway.igw.id
}

resource "aws_route" "route_to_nat" {
  route_table_id = aws_route_table.nat_private_route_table.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id = aws_nat_gateway.nat.id
}


# Add routing associations to subnet
resource "aws_route_table_association" "association_for_route_to_igw" { 
  count = 2
  route_table_id = aws_route_table.igw_private_route_table.id
  subnet_id = aws_subnet.app_subnet[count.index].id
}

resource "aws_route_table_association" "association_for_route_to_nat" {  
  count = 2
  route_table_id = aws_route_table.nat_private_route_table.id
  subnet_id = aws_subnet.db_subnet[count.index].id
}

#creating security group

resource "aws_security_group" "app_sg" {
  name        = "web-app-sg"
  description = "allow app"
  vpc_id      = aws_vpc.main.id
  tags = {
    Name = "web-app-sg"
  }
 }


resource "aws_security_group_rule" "r1" {
  type              = "ingress"
  from_port         = 80
  to_port           = 80
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.app_sg.id
}

resource "aws_security_group_rule" "r2" {
  type              = "ingress"
  from_port         = 465
  to_port           = 465
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.app_sg.id
}

resource "aws_security_group_rule" "r3" {
  type              = "ingress"
  from_port         = 587
  to_port           = 587
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.app_sg.id
}

resource "aws_security_group_rule" "r4" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.app_sg.id
}

resource "aws_security_group_rule" "r5" {
  type              = "ingress"
  from_port         = 3306
  to_port           = 3306
  protocol          = "tcp"
  source_security_group_id = aws_security_group.db_sg.id
  security_group_id = aws_security_group.app_sg.id
}

resource "aws_security_group" "db_sg" {
  name        = "db-sg"
  description = "db security group"
  vpc_id      = aws_vpc.main.id
  tags = {
    Name = "db-sg"
  }
}

resource "aws_security_group_rule" "db_r1" {
  type              = "ingress"
  from_port         = 3306
  to_port           = 3306
  protocol          = "tcp"
  source_security_group_id = aws_security_group.app_sg.id
  security_group_id = aws_security_group.db_sg.id
}

resource "aws_security_group_rule" "db_r2" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.db_sg.id
}


resource "aws_opsworks_stack" "demo" {
  name                         = "LSEG_Demo2"
  region                       = "ap-south-1"
  service_role_arn             = "arn:aws:iam::886038311436:role/aws-opsworks-service-role"
  default_instance_profile_arn = "arn:aws:iam::886038311436:instance-profile/aws-opsworks-ec2-role"
  configuration_manager_version = "14"
  default_os = "Amazon Linux 2018.03"
  use_custom_cookbooks = true
  default_root_device_type= "ebs"
  vpc_id =  aws_vpc.main.id
  default_subnet_id = aws_subnet.app_subnet[0].id

  custom_cookbooks_source{
    type = "git"
    url = "https://github.com/sdr077/AppMonitor.git"
    #ssh_key= ""


  }

}

resource "aws_opsworks_custom_layer" "application" {
  name       = "Application Layer"
  short_name = "app"
  stack_id   = aws_opsworks_stack.demo.id
  custom_security_group_ids = [aws_security_group.app_sg.id]
  custom_setup_recipes = ["configuration::configFile"]
  auto_assign_public_ips = true


}

resource "aws_opsworks_instance" "app1" {
  stack_id = aws_opsworks_stack.demo.id

  layer_ids = [
    aws_opsworks_custom_layer.application.id,
  ]

  instance_type = "t3.micro"
  os            = "Amazon Linux 2018.03"
  state         = "running"
}




resource "aws_opsworks_custom_layer" "db" {
  name       = "DB Layer"
  short_name = "db"
  stack_id   = aws_opsworks_stack.demo.id
  custom_security_group_ids = [aws_security_group.db_sg.id]
  auto_assign_public_ips = true
}

resource "aws_opsworks_instance" "db1" {
  stack_id = aws_opsworks_stack.demo.id

  layer_ids = [
    aws_opsworks_custom_layer.db.id,
  ]

  instance_type = "t3.micro"
  os            = "Amazon Linux 2018.03"
  state         = "running"
  subnet_id = aws_subnet.db_subnet[0].id
}


```

#### Configuration using chef #####

```
configFile.rb

execute 'creating html folder' do
	command 'mkdir -p /var/www/html'
end

package 'httpd' do
	action :install
end

service 'httpd' do
  action [ :enable, :start ]
  supports :reload => true, :restart => true, :status => true, :stop => true
end

```

```
mysql_conf.rb

#copy mysql.sh script to sevrver
cookbook_file '/root/mysql.sh' do
	source 'mysql.sh'
	owner 'root'
	group 'root'
	mode '0744'
end

execute "Install and configure mysql" do
	command "bash /root/mysql.sh"
end

```

```

mysql.sh

#!/bin/bash

#Install mysql server
yum install mysql-server -y 

# Enable auto start
chkconfig mysqld on

#Start mysql server
service mysqld start

```

