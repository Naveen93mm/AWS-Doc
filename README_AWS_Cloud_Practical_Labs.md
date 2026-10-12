# AWS Cloud Practical Labs

A hands-on, console-first guide for learning core AWS services. Each lab explains the objective, console navigation, important configuration choices, validation steps, and cleanup. Use the code examples as starting points and adapt names, regions, and resource IDs to your own account.

> **Important:** AWS Console labels and available options can change. Your account may show slightly different screens. This guide uses the general AWS Management Console workflow.
>
> **Cost and security:** Some resources incur charges even when idle, including NAT Gateways, load balancers, RDS databases, public IPv4 addresses, and provisioned storage. Use a training account, choose the smallest suitable configuration, and delete lab resources when finished. Never publish access keys, private keys, passwords, account IDs, or other secrets in GitHub.

## Table of Contents

- [0. Lab Preparation](#0-lab-preparation)
- [1. Amazon EC2](#1-amazon-ec2)
- [2. Amazon EBS Volumes and Snapshots](#2-amazon-ebs-volumes-and-snapshots)
- [3. Amazon S3](#3-amazon-s3)
- [4. IAM: Users, Groups, and Roles](#4-iam-users-groups-and-roles)
- [5. Amazon VPC Networking](#5-amazon-vpc-networking)
- [6. Elastic Load Balancing](#6-elastic-load-balancing)
- [7. EC2 Auto Scaling](#7-ec2-auto-scaling)
- [8. Amazon RDS](#8-amazon-rds)
- [9. Amazon DynamoDB](#9-amazon-dynamodb)
- [10. Amazon SNS Email Notifications](#10-amazon-sns-email-notifications)
- [11. Amazon SQS](#11-amazon-sqs)
- [12. Amazon CloudWatch Dashboards and Alarms](#12-amazon-cloudwatch-dashboards-and-alarms)
- [13. AWS CloudTrail](#13-aws-cloudtrail)
- [14. AWS Lambda: S3 Upload to SNS Email](#14-aws-lambda-s3-upload-to-sns-email)
- [15. Final Review and Cleanup Checklist](#15-final-review-and-cleanup-checklist)

---

## 0. Lab Preparation

### Requirements

- An AWS account with permission to create the resources in these labs.
- Access to the AWS Management Console.
- A selected AWS Region. Keep related resources in the same Region unless the lab explicitly requires otherwise.
- A browser and, for EC2 connection exercises, an SSH client or the EC2 Instance Connect/Systems Manager option available to your instance.
- A working email address for SNS subscription confirmation.
- A GitHub repository for notes and code.

### Naming convention

Use clear names so resources are easy to identify and remove. Example names:

- EC2 instance: `aws-lab-web-01`
- Security group: `aws-lab-web-sg`
- S3 bucket: `aws-lab-unique-name-12345`
- SNS topic: `aws-lab-file-notifications`
- Lambda function: `aws-lab-s3-upload-notifier`

S3 bucket names must be globally unique. Replace every example name with one that is available.

### General console workflow

1. Sign in to the AWS Management Console.
2. Select the required Region from the Region selector.
3. Open the service by searching for its name in the console search bar.
4. Select **Create**, **Create resource**, or the equivalent action.
5. Configure required fields and review optional settings.
6. Review the summary carefully, then create the resource.
7. Open the resource details and verify its status and configuration.
8. Record non-secret resource names and IDs in your lab notes.
9. Delete resources after practice if they are no longer needed.

### Account safety

- Prefer an IAM role or IAM Identity Center access rather than creating long-lived access keys for daily use.
- Enable MFA for privileged users.
- Do not use the root user for routine work.
- Apply least-privilege permissions. Avoid attaching `AdministratorAccess` to normal lab users.
- Do not commit `.pem` files, credentials, `.env` files, or real customer data to GitHub.
- Check **Billing and Cost Management** after cost-generating labs.

---

## 1. Amazon EC2

**Objective:** Launch and connect to a virtual server in AWS.

### Create an EC2 instance

1. Open **EC2** in the AWS Management Console.
2. Choose **Instances** from the left navigation.
3. Select **Launch instances**.
4. Under **Name and tags**, enter a name such as `aws-lab-web-01`. Add tags if needed, for example `Environment=Lab` and `Owner=Student`.
5. Under **Application and OS Images (Amazon Machine Image / AMI)**, select an operating system. For a beginner Linux lab, choose a suitable Amazon Linux AMI.
6. Check the **Architecture** shown for the AMI, such as 64-bit x86 or 64-bit Arm. Select software and instance types that support the same architecture.
7. Under **Instance type**, choose a small instance type appropriate for the lab and your account's Free Tier eligibility. Availability and Free Tier rules vary by account and date.
8. Under **Key pair (login)**, select an existing key pair or choose to create one. If creating a new key pair, download the private key and store it securely. You cannot download the same private key again later.
9. Under **Network settings**, choose **Edit** to customize settings when needed.
   - Select the intended VPC.
   - Select a subnet.
   - Decide whether the instance needs a public IP address.
   - Select an existing security group or create one.
10. Configure the security group with only the required inbound traffic. For a basic web server, allow HTTP (TCP 80) from the intended audience. For SSH (TCP 22), restrict the source to your own public IP address rather than `0.0.0.0/0`.
11. Under **Configure storage**, review the root EBS volume size, volume type, and deletion behavior. Increase storage only when the exercise needs it.
12. Expand **Advanced details** only if the lab requires an IAM instance profile, user data, or other advanced settings. Do not paste secrets into user data.
13. Review the **Summary** panel, including Region, AMI, instance type, network, security group, and storage.
14. Select **Launch instance**.
15. Choose **View all instances** and wait until the instance state is **Running** and the status checks pass.

### Connect to the instance

1. In **EC2 → Instances**, select the instance.
2. Select **Connect**.
3. Use a connection method supported by the instance and account, such as **EC2 Instance Connect**, **Session Manager**, or an SSH client.
4. For SSH, ensure the instance is reachable, the security group allows TCP 22 from your IP, and the correct private key is used. On Unix-like systems, set restrictive permissions on the key file if required, for example `chmod 400 my-key.pem`.
5. For Amazon Linux, the usual SSH user is `ec2-user`; other AMIs may use a different default username.

### Optional: install a web server on Amazon Linux

Connect to the instance and run:

```bash
sudo dnf update -y
sudo dnf install -y nginx
sudo systemctl enable --now nginx
echo "Hello from my AWS EC2 lab" | sudo tee /usr/share/nginx/html/index.html
```

If the selected AMI uses a different package manager, adapt the commands. In a browser, open `http://PUBLIC-IP-ADDRESS` only if the instance has a reachable public address and the security group permits inbound TCP 80.

### Verify

- Instance state is **Running**.
- Status checks pass.
- Instance details show the expected AMI, architecture, instance type, subnet, and security group.
- If configured as a web server, the test page loads from an allowed network.

### Common issues

- **Cannot connect:** Check route table, subnet, public IP, security group, network ACL, key pair, and instance status.
- **Web page times out:** Confirm the web server is running and inbound TCP 80 is allowed.
- **Architecture mismatch:** Select a compatible AMI and instance type.

### Cleanup

Select the instance, choose **Instance state → Terminate instance**, and confirm. Termination is destructive. Review attached EBS volumes and other related resources separately.

---

## 2. Amazon EBS Volumes and Snapshots

Amazon EBS provides block storage for EC2. A volume is attached to an instance; a snapshot is a point-in-time backup of a volume.

### Create an additional EBS volume

1. Open **EC2 → Elastic Block Store → Volumes**.
2. Select **Create volume**.
3. Choose a volume type, such as `gp3`, and set a size appropriate for the lab.
4. Select the same Availability Zone as the target EC2 instance. An EBS volume can normally be attached only to an instance in the same Availability Zone.
5. Select **Create volume**.
6. Wait until the volume state is **Available**.

### Attach the volume to an EC2 instance

1. Select the new volume.
2. Choose **Actions → Attach volume**.
3. Select the target instance in the same Availability Zone.
4. Review the device name and choose **Attach volume**.
5. Connect to the instance and identify the new disk before making changes.

On a Linux instance, inspect devices first:

```bash
lsblk
```

**Warning:** The next commands create a new filesystem. Run them only against the confirmed new, empty disk. Formatting the wrong device can destroy data.

Example only; replace `/dev/nvme1n1` with the correct device shown by `lsblk`:

```bash
sudo file -s /dev/nvme1n1
# Only if this is the new empty disk and it has no filesystem:
sudo mkfs -t xfs /dev/nvme1n1
sudo mkdir -p /data
sudo mount /dev/nvme1n1 /data
df -h
```

For a persistent mount, obtain the filesystem UUID with `sudo blkid`, then add a carefully verified UUID-based entry to `/etc/fstab`. Test with `sudo mount -a` before rebooting. Device names can differ between instance families; NVMe-based instances commonly expose EBS devices as NVMe names.

### Create an EBS snapshot

1. Open **EC2 → Volumes**.
2. Select the volume to back up.
3. Choose **Actions → Create snapshot**.
4. Enter a description, such as `aws-lab-data-volume-backup`.
5. Add tags if useful.
6. Select **Create snapshot**.
7. Open **EC2 → Elastic Block Store → Snapshots** and wait for the snapshot to complete.

For application-consistent backups, coordinate with the application or filesystem before taking a snapshot. Snapshots are incremental at the storage level, but they can still incur charges.

### Restore a volume from a snapshot

1. Open **Snapshots** and select the required completed snapshot.
2. Choose **Actions → Create volume from snapshot**.
3. Select the Availability Zone where the target instance runs.
4. Review volume type, size, encryption, and tags.
5. Create the volume, then attach it using the steps above.

### Verify and clean up

Confirm the volume is attached to the intended instance and the filesystem is mounted. When finished, unmount the filesystem before detaching the volume. Delete only disposable lab volumes and snapshots that are no longer required.

---

## 3. Amazon S3

**Objective:** Store and retrieve objects using S3 buckets.

### Create a general purpose bucket

1. Open **Amazon S3**.
2. Choose **Buckets → Create bucket**.
3. Enter a globally unique bucket name using lowercase letters, numbers, and hyphens.
4. Choose the AWS Region.
5. Keep **Block all public access** enabled unless a carefully reviewed exercise explicitly requires public access. Do not make a bucket public just to test uploads.
6. Keep the recommended object ownership setting, normally **Bucket owner enforced**, unless the lab requires a different ownership model.
7. Leave versioning disabled for a simple introductory lab, or enable it to practice object versioning.
8. Review default encryption settings. S3 encrypts new objects at rest by default; choose an approved encryption option if the lab requires it.
9. Review optional settings and tags.
10. Select **Create bucket**.

### Upload and download an object

1. Open the bucket.
2. Select **Upload → Add files** or **Add folder**.
3. Choose a sample file that contains no sensitive information.
4. Review upload settings and choose **Upload**.
5. Select the object to view its details.
6. Choose **Download** to retrieve it.

### Create a folder-like prefix (directory)

S3 is object storage and does not use traditional filesystem directories. A “folder” in the console is generally a key prefix.

1. Open the bucket.
2. Select **Create folder**.
3. Enter a name such as `documents`.
4. Choose **Create folder**.
5. Open the prefix and upload an object, or upload an object with a key such as `documents/example.txt`.

An S3 **directory bucket** is a separate S3 bucket type associated with S3 Express One Zone and has different supported features and requirements. Do not confuse it with a folder inside a general purpose bucket. For this class, create a general purpose bucket and a folder-like prefix unless the instructor specifically requests an S3 directory bucket lab.

### Enable versioning (optional)

1. Open the bucket and select **Properties**.
2. Find **Bucket Versioning** and choose **Edit**.
3. Enable versioning and save.
4. Upload a changed file with the same object key.
5. Review the object's versions to see the separate versions.

### Verify and cleanup

Confirm the bucket is in the intended Region and the object appears in the list. Delete test objects first, then delete the empty bucket. If versioning is enabled, remove test object versions and delete markers as needed before deleting the bucket.

---

## 4. IAM: Users, Groups, and Roles

IAM controls who can access AWS and which actions they can perform. Follow least privilege.

### Create an IAM group

1. Open **IAM → User groups**.
2. Choose **Create group**.
3. Enter a name such as `aws-lab-readonly-group`.
4. Attach only the policy required for the lab. For a basic viewing exercise, consider an appropriate AWS-managed read-only policy, but review its scope first.
5. Create the group.

### Create an IAM user and add it to a group

1. Open **IAM → Users → Create user**.
2. Enter a username such as `student-lab-user`.
3. Choose console access only if the student needs to sign in to the AWS Console. Prefer centrally managed access through IAM Identity Center for real organizations.
4. Add the user to the intended group.
5. Review permissions and create the user.
6. Follow the console's secure sign-in instructions. Share temporary access details through an approved private channel, never in a public repository.

Avoid creating programmatic access keys unless the lab specifically requires them. Never commit keys to GitHub.

### Create an IAM role

1. Open **IAM → Roles → Create role**.
2. Choose the trusted entity type that matches the use case, such as **AWS service**.
3. Select the service that will assume the role, for example **Lambda** or **EC2**.
4. Attach the minimum permissions needed by that service. For Lambda, this often includes a basic execution policy for CloudWatch Logs plus specific access to the resources it uses.
5. Enter a clear role name, such as `aws-lab-lambda-execution-role`.
6. Review the trust relationship and permissions, then create the role.
7. Select the role later from the relevant service's execution-role or instance-profile setting.

### Verify

Open the user, group, or role details and review attached policies and trust relationships. Test only with authorized accounts and non-sensitive resources. If access is denied, inspect the exact action, resource ARN, identity policy, resource policy, permissions boundary, and any organization-level restrictions.

### Cleanup

Remove lab users, groups, roles, and policies when no longer needed. Do not delete a role that is still used by a live resource.

---

## 5. Amazon VPC Networking

A VPC is a logically isolated network. Subnets belong to Availability Zones; route tables determine where traffic is sent.

### Create a VPC and subnets

1. Open **VPC → Your VPCs → Create VPC**.
2. Choose **VPC only** for a manual networking lab, or **VPC and more** for a guided setup.
3. Enter a name such as `aws-lab-vpc`.
4. Enter an IPv4 CIDR block, for example `10.20.0.0/16`, ensuring it does not overlap with networks you need to connect.
5. Create the VPC.
6. Open **Subnets → Create subnet**.
7. Select the VPC.
8. Create a public subnet CIDR such as `10.20.1.0/24` in one Availability Zone and, if required, a private subnet such as `10.20.2.0/24` in another or the same Availability Zone.
9. Create the subnets.
10. Enable auto-assign public IPv4 only for a subnet where that behavior is intended. A public IP alone does not make a subnet public; routing and security controls must also allow connectivity.

### Create and attach an Internet Gateway

1. Open **VPC → Internet gateways → Create internet gateway**.
2. Enter a name tag and create it.
3. Select the gateway and choose **Actions → Attach to a VPC**.
4. Select `aws-lab-vpc` and attach it.

### Create a route table and associate a public subnet

1. Open **VPC → Route tables → Create route table**.
2. Enter a name and select the VPC.
3. Create the route table.
4. Select it and open **Subnet associations → Edit subnet associations**.
5. Select the intended public subnet and save.
6. Open **Routes → Edit routes → Add route**.
7. Set destination `0.0.0.0/0` and target the Internet Gateway.
8. Save routes.

A subnet is considered public when its route table sends internet-bound IPv4 traffic to an Internet Gateway. Instances still need suitable addressing and security rules.

### Create a NAT Gateway (cost-generating)

A NAT Gateway allows resources in private subnets to initiate supported outbound IPv4 connections without accepting unsolicited inbound internet connections.

1. Open **VPC → NAT gateways → Create NAT gateway**.
2. Choose the public subnet where the NAT Gateway should reside.
3. Allocate or select an Elastic IP as prompted.
4. Create the NAT Gateway and wait until its state is **Available**.
5. Open the route table associated with the private subnet.
6. Add route `0.0.0.0/0` with the NAT Gateway as target.
7. Save the route.

**Cost note:** NAT Gateways have hourly and data-processing charges. Delete the NAT Gateway and release its unused Elastic IP after the lab.

### Create a VPC peering connection

1. Open **VPC → Peering connections → Create peering connection**.
2. Enter a name.
3. Choose the requester VPC and the accepter VPC. They can be in the same or different accounts/Regions where supported.
4. Create the peering connection.
5. If it is cross-account, sign in to the accepter account and accept the request.
6. In each VPC's relevant route table, add routes to the other VPC's CIDR with the peering connection as target.
7. Check security groups and network ACLs on both sides.

VPC peering is not transitive. One peering connection does not automatically route traffic through a third VPC.

### Verify

Review VPC CIDRs, subnet associations, routes, gateway states, and security rules. Test connectivity only between authorized lab resources. Route tables alone do not override security groups or network ACLs.

### Cleanup

Terminate instances in the VPC first. Delete dependent resources in the correct order: NAT Gateways, peering connections, route table associations/routes, subnets, Internet Gateway detachment/deletion, and finally the VPC. Some resources may prevent deletion until dependencies are removed.

---

## 6. Elastic Load Balancing

**Objective:** Distribute incoming traffic across multiple targets. This lab uses an Application Load Balancer (ALB) for HTTP/HTTPS traffic.

### Prerequisites

- A VPC with at least two subnets in different Availability Zones for a typical highly available ALB setup.
- Two EC2 instances running a web server, ideally in separate Availability Zones.
- A security group for the ALB and a security group for the instances.

### Create a target group

1. Open **EC2 → Target Groups → Create target group**.
2. Choose **Instances** as the target type for this lab.
3. Enter a name such as `aws-lab-web-targets`.
4. Select **HTTP** and port `80`.
5. Select the VPC that contains the instances.
6. Configure the health check path, usually `/` for a basic web server.
7. Create the target group.
8. Register the EC2 instances and include them as targets.

### Create an Application Load Balancer

1. Open **EC2 → Load Balancers → Create load balancer**.
2. Choose **Application Load Balancer**.
3. Enter a name such as `aws-lab-alb`.
4. Set the scheme to **Internet-facing** only if public access is intended; otherwise use an internal load balancer.
5. Choose the VPC and select subnets in at least two Availability Zones.
6. Select or create an ALB security group that allows inbound HTTP on port 80 from the intended client network.
7. Configure a listener for HTTP port 80 and forward to `aws-lab-web-targets`.
8. Review and create the load balancer.

### Configure instance security

The instance security group should allow inbound HTTP from the **ALB security group**, not broadly from the entire internet. The ALB security group must allow client traffic on the listener port.

### Verify

1. Wait for the load balancer state to become **Active**.
2. Open the target group and confirm the targets are **Healthy**.
3. Copy the ALB DNS name from its details.
4. Open `http://ALB-DNS-NAME` in a browser.
5. Refresh several times to observe responses from registered targets. Give each test instance a different index page so you can identify which target responds.

### Cleanup

Delete the ALB and target group after testing, and terminate lab instances if they are no longer needed. Load balancers incur charges while provisioned.

---

## 7. EC2 Auto Scaling

**Objective:** Automatically maintain the desired number of EC2 instances and scale capacity.

### Create a launch template

1. Open **EC2 → Launch Templates → Create launch template**.
2. Enter a name such as `aws-lab-web-template`.
3. Select a suitable AMI and instance type.
4. Choose a key pair only if direct SSH access is required.
5. Select a security group that permits only the required traffic.
6. Configure storage and any non-secret user data needed to install the web server.
7. Create the launch template.

Example user data for a compatible Amazon Linux AMI:

```bash
#!/bin/bash
dnf update -y
dnf install -y nginx
systemctl enable --now nginx
echo "Hello from Auto Scaling" > /usr/share/nginx/html/index.html
```

User data is executed on first boot by default on common Linux AMIs. Do not place credentials or secrets in it.

### Create an Auto Scaling group

1. Open **EC2 → Auto Scaling Groups → Create Auto Scaling group**.
2. Enter a name such as `aws-lab-asg`.
3. Select the launch template and version.
4. Select the VPC and subnets in at least two Availability Zones.
5. If using an ALB, attach the target group created in the load-balancing lab.
6. Set desired capacity, minimum capacity, and maximum capacity. For a small demo, values such as desired `2`, minimum `1`, maximum `3` may be appropriate, subject to budget and quotas.
7. Configure health checks. Use ELB health checks when the group is attached to a load balancer and you want unhealthy targets replaced based on load balancer health.
8. Add a scaling policy, such as target tracking for average CPU utilization, if the exercise requires automatic scaling.
9. Review and create the group.

### Verify

Open the Auto Scaling group and review **Activity**, **Instance management**, and desired/minimum/maximum capacity. Confirm instances launch and register with the target group. You can change desired capacity to observe scale-out/scale-in, but remember that manual changes can be overridden by scaling policies.

### Cleanup

Set the group to a low desired capacity if needed, then delete the Auto Scaling group. Remove the launch template and load-balancing resources only when they are no longer required. Monitor the group to ensure it does not recreate instances after you think you have cleaned up.

---

## 8. Amazon RDS

**Objective:** Create a managed relational database.

### Create a database

1. Open **Amazon RDS → Databases → Create database**.
2. Choose **Standard create** to review the available options.
3. Select a database engine such as MySQL or PostgreSQL.
4. Choose a template suitable for a learning lab. Avoid production or Multi-AZ options unless the exercise requires them.
5. Enter a DB instance identifier, such as `aws-lab-db`.
6. Set the master username and a strong password. Store it securely; do not place it in GitHub or source code.
7. Choose an instance class and storage size suitable for the lab.
8. Under connectivity, choose the intended VPC and DB subnet group.
9. For a safer default, set **Public access** to **No** and connect from an authorized EC2 instance in the VPC.
10. Configure the VPC security group to allow the database port only from the application or EC2 security group. Common defaults are MySQL `3306` and PostgreSQL `5432`.
11. Review backup retention, encryption, monitoring, and other settings.
12. Create the database and wait until its status is **Available**.
13. Open the database details and copy the endpoint and port.

### Connect and verify

From an authorized client in the VPC, connect using the appropriate database client and credentials. Ensure the database security group permits the client's security group. Do not expose a database port to the whole internet for a beginner lab.

### Cleanup

Take a final snapshot only if needed. Delete the database and review the final snapshot and automated backup choices carefully. Retained snapshots and backups may continue to incur charges.

---

## 9. Amazon DynamoDB

**Objective:** Create a managed NoSQL table.

### Create a table

1. Open **DynamoDB → Tables → Create table**.
2. Enter a table name such as `aws-lab-students`.
3. Enter a partition key, for example `studentId`, with type **String**.
4. Add a sort key only if the data model requires it, such as `createdAt`.
5. Select the capacity mode appropriate for the lab. On-demand capacity is simple for unpredictable practice workloads, but usage can still incur charges.
6. Review encryption, point-in-time recovery, and tags.
7. Choose **Create table** and wait until the table is **Active**.

### Add and query an item

1. Open the table and select **Explore table items**.
2. Choose **Create item**.
3. Add an item such as:

```json
{
  "studentId": "STU001",
  "name": "Example Student",
  "course": "AWS"
}
```

4. Save the item.
5. Use the table's item explorer to query or scan sample records. Prefer **Query** when you know the partition key; scans examine the table and can be inefficient on large tables.

### Verify and cleanup

Confirm the table is **Active** and the item can be retrieved. Delete the table after the lab if it is no longer needed.

---

## 10. Amazon SNS Email Notifications

**Objective:** Send an email notification using an SNS topic.

### Create an SNS topic

1. Open **Amazon SNS → Topics → Create topic**.
2. Select **Standard** for a general email-notification lab.
3. Enter a topic name such as `aws-lab-file-notifications`.
4. Review optional settings and permissions.
5. Choose **Create topic**.
6. Open the topic details and copy the topic ARN for use in the Lambda lab.

### Subscribe an email address

1. Open the topic and choose **Create subscription**.
2. Select protocol **Email**.
3. Enter an email address you control.
4. Create the subscription.
5. Open your inbox and find the AWS subscription confirmation email.
6. Click **Confirm subscription**.
7. Return to SNS and verify the subscription status is **Confirmed**.

### Publish a test message

1. Open the topic.
2. Choose **Publish message**.
3. Enter a subject such as `AWS SNS Lab Test`.
4. Enter a short message body.
5. Publish the message.
6. Check the subscribed inbox and spam folder.

### Cleanup

Delete the test topic and subscriptions if no longer required. Do not send unsolicited email; use only addresses whose owners have agreed to receive the messages.

---

## 11. Amazon SQS

**Objective:** Create a queue for asynchronous message processing.

### Create a queue

1. Open **Amazon SQS → Create queue**.
2. Choose **Standard** for a general-purpose introductory lab, or **FIFO** when ordered, deduplicated processing is required.
3. Enter a name such as `aws-lab-orders-queue`.
4. Review visibility timeout, message retention, delivery delay, and receive wait time.
5. For a Standard queue, review the default at-least-once delivery behavior and design consumers to tolerate duplicate messages.
6. Keep encryption enabled using the supported default unless the lab has a specific key requirement.
7. Choose **Create queue**.

### Send and receive a test message

1. Open the queue.
2. Choose **Send and receive messages**.
3. Enter a sample message body, such as `Hello from the SQS lab`.
4. Send the message.
5. Choose **Poll for messages** or the equivalent receive action.
6. Open the received message to inspect its body.
7. Delete the message after processing to prevent it from being received again.

### Verify and cleanup

Confirm the queue is active and the message can be sent and received. Delete the queue after the exercise if it is no longer needed.

---

## 12. Amazon CloudWatch Dashboards and Alarms

### Create a dashboard

1. Open **CloudWatch → Dashboards**.
2. Choose **Create dashboard**.
3. Enter a name such as `aws-lab-dashboard`.
4. Choose a widget type, such as a line or number widget.
5. Select a metric source, for example **EC2 → Per-Instance Metrics → CPUUtilization**.
6. Select the instance and metric, then add it to the widget.
7. Repeat for other metrics if useful.
8. Save the dashboard.

Metric availability depends on the service and configuration. EC2 basic monitoring provides common host metrics; detailed monitoring and in-guest memory/disk metrics may require additional configuration or the CloudWatch Agent.

### Create a CloudWatch alarm

1. Open **CloudWatch → Alarms → All alarms**.
2. Choose **Create alarm**.
3. Select **Select metric**.
4. Choose the namespace and metric, such as **EC2 → Per-Instance Metrics → CPUUtilization**.
5. Select the instance and metric.
6. Set a threshold and evaluation periods appropriate for the exercise.
7. Configure missing-data handling deliberately.
8. For alarm notification, select an existing SNS topic or create one.
9. Give the alarm a name such as `aws-lab-high-cpu`.
10. Review and create the alarm.

### Test and verify

Generate a controlled workload only on a disposable lab instance if testing a CPU alarm. Review the alarm state and its history. A new alarm may show **Insufficient data** until enough metric samples arrive.

### Cleanup

Delete test alarms and dashboards when no longer needed. Review any SNS topic created for notifications.

---

## 13. AWS CloudTrail

**Objective:** Record account activity for auditing and investigation.

### View recent event history

1. Open **CloudTrail → Event history**.
2. Select a time range and filter by event name, user, resource, or other available fields.
3. Open an event to inspect details such as the event source, identity, time, and request parameters.
4. Event history provides recent management events for the selected Region. A trail is needed for ongoing delivery of logs to a storage destination.

### Create a trail and store logs in S3

1. Open **CloudTrail → Trails → Create trail**.
2. Enter a name such as `aws-lab-management-trail`.
3. Choose whether the trail applies to one Region or all Regions. For a broad audit trail, consider all Regions.
4. Create a new S3 bucket for logs or select an existing dedicated logging bucket.
5. Review log file validation and encryption settings.
6. Configure management events. For a basic lab, ensure read and write management events are included as intended.
7. Review optional data events carefully. S3 object-level data events and Lambda data events can create additional logging charges and must be explicitly configured when needed.
8. Create the trail.
9. Generate a few non-sensitive test management events, such as viewing a resource.
10. After delivery time, inspect the S3 bucket for CloudTrail log files.

### Verify

Confirm the trail is logging and the S3 destination is correct. Use Event history for quick recent lookups and the S3-delivered trail files for longer-term retention and analysis.

### Cleanup

If the trail is only for practice, stop or delete it after the lab. Delete its log bucket only after confirming that the logs are disposable and no retention requirement applies. CloudTrail logging and storage may incur charges depending on configuration and usage.

---

## 14. AWS Lambda: S3 Upload to SNS Email

**Objective:** Automatically send an SNS email notification whenever an object is uploaded to a specific S3 bucket. This lab uses Python and an S3 event notification.

### Architecture

`User uploads object → S3 ObjectCreated event → Lambda function → SNS topic → Confirmed email subscriber`

### Step A: Create an SNS topic and confirm an email subscription

Follow [Amazon SNS Email Notifications](#10-amazon-sns-email-notifications). Keep the topic ARN ready, and ensure the email subscription is **Confirmed**.

### Step B: Create an S3 bucket for uploads

1. Open **S3 → Create bucket**.
2. Enter a globally unique name, such as `aws-lab-upload-source-12345`.
3. Select the Region where you will create the Lambda function and SNS topic.
4. Keep **Block all public access** enabled.
5. Create the bucket.
6. Do not configure a public bucket policy. The event notification does not require public access.

### Step C: Create a Lambda execution role

1. Open **IAM → Roles → Create role**.
2. Choose **AWS service** as the trusted entity and **Lambda** as the use case.
3. Attach `AWSLambdaBasicExecutionRole` so the function can write logs to CloudWatch Logs.
4. Create the role, for example `aws-lab-s3-sns-lambda-role`.
5. Open the role and choose **Add permissions → Create inline policy**.
6. In the JSON editor, add an SNS publish permission scoped to your topic ARN. Replace `REGION`, `ACCOUNT_ID`, and `TOPIC_NAME` with your actual values:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublishToLabTopic",
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:REGION:ACCOUNT_ID:TOPIC_NAME"
    }
  ]
}
```

7. Name the policy `aws-lab-publish-to-sns` and save it.

Use the exact topic ARN copied from SNS. Do not grant `sns:*` on all resources for this lab.

### Step D: Create the Lambda function

1. Open **AWS Lambda → Functions → Create function**.
2. Select **Author from scratch**.
3. Enter the function name `aws-lab-s3-upload-notifier`.
4. Select a supported Python runtime available in the console.
5. Under **Permissions**, choose **Use an existing role** and select `aws-lab-s3-sns-lambda-role`.
6. Create the function.
7. In **Configuration → Environment variables**, choose **Edit**.
8. Add an environment variable:
   - Key: `SNS_TOPIC_ARN`
   - Value: the full SNS topic ARN
9. Save the environment variables.

Environment variables are convenient for non-secret configuration such as an ARN. Do not store passwords or API secrets in source code or ordinary environment variables.

### Step E: Add the Python code

In the Lambda **Code** tab, replace the starter code with the following and choose **Deploy**:

```python
import json
import os
import urllib.parse
import boto3

sns = boto3.client("sns")
SNS_TOPIC_ARN = os.environ["SNS_TOPIC_ARN"]


def lambda_handler(event, context):
    records = event.get("Records", [])
    sent = 0

    for record in records:
        if record.get("eventSource") != "aws:s3":
            continue

        bucket = record["s3"]["bucket"]["name"]
        key = urllib.parse.unquote_plus(record["s3"]["object"]["key"])
        size = record["s3"]["object"].get("size", 0)
        event_name = record.get("eventName", "ObjectCreated")

        subject = "S3 Upload Notification"
        message = (
            "A new object event was received.\n\n"
            f"Event: {event_name}\n"
            f"Bucket: {bucket}\n"
            f"Object key: {key}\n"
            f"Size: {size} bytes\n"
            f"Region: {record.get('awsRegion', 'unknown')}\n"
        )

        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject=subject,
            Message=message
        )
        sent += 1

    return {
        "statusCode": 200,
        "body": json.dumps({
            "message": "Processed S3 event records",
            "notifications_sent": sent
        })
    }
```

The Lambda Python runtime includes `boto3` in standard AWS-managed runtimes, but runtime packaging can change. If the selected runtime or deployment package does not include a required SDK version, package it with the function or a Lambda layer.

### Step F: Add the S3 trigger

1. Open the Lambda function's **Configuration → Triggers** section or choose **Add trigger** from the function overview.
2. Select **S3** as the source.
3. Select the upload bucket created in Step B.
4. Choose the event type **All object create events** (or a narrower event type for the lab).
5. Optionally enter a prefix such as `uploads/` or a suffix such as `.txt` to limit triggering.
6. Acknowledge the recursive invocation warning if displayed, and make sure the Lambda function does **not** write output objects back to the same triggering prefix. This example only publishes an SNS message and does not write to S3.
7. Add the trigger.

AWS normally configures the bucket's permission to invoke the function through the console workflow. S3 event notifications and the Lambda function must be in a supported Region configuration; keeping them in the same Region is the simplest lab setup.

### Step G: Test the complete workflow

1. Confirm the SNS email subscription is **Confirmed**.
2. Open the upload bucket and choose **Upload**.
3. Upload a harmless test file such as `hello.txt`.
4. Wait briefly and check the subscribed email inbox and spam folder.
5. Open **Lambda → Monitor → View CloudWatch logs** to inspect the invocation.
6. If the email does not arrive, inspect the Lambda error logs, SNS topic ARN environment variable, execution-role permissions, S3 trigger configuration, and subscription status.

### Troubleshooting

- **AccessDenied on `sns:Publish`:** Verify the execution role policy permits `sns:Publish` to the exact topic ARN.
- **No Lambda invocation:** Check the trigger is attached to the correct bucket and event type; confirm the object was uploaded after the trigger was created.
- **Lambda runs but no email arrives:** Confirm the SNS subscription is confirmed and the function publishes to the correct topic.
- **Malformed object key:** S3 event keys are URL encoded; the code uses `unquote_plus` to decode them.
- **Duplicate email:** S3 event notifications are delivered at least once and may occasionally be duplicated. Design downstream workflows to tolerate duplicate events if needed.
- **Large upload or many files:** Consider notification volume, SNS quotas, and cost. This sample sends one email per event record.

### Cleanup

1. Remove the S3 event notification trigger.
2. Delete the Lambda function.
3. Delete the IAM role and its lab-only policy if not used elsewhere.
4. Delete test objects and the upload bucket.
5. Delete the SNS topic and subscription if no longer needed.
6. Review CloudWatch log groups and delete lab logs only if they are no longer needed.

---

## 15. Final Review and Cleanup Checklist

Use this checklist at the end of every class:

- [ ] EC2 instances are stopped or terminated as intended.
- [ ] Unused EBS volumes and snapshots are removed.
- [ ] S3 test objects, versions, and buckets are removed when appropriate.
- [ ] Unused Elastic IP addresses are released.
- [ ] NAT Gateways are deleted after private-subnet exercises.
- [ ] Load balancers and target groups are removed after testing.
- [ ] Auto Scaling groups are deleted or configured so they will not recreate instances.
- [ ] RDS instances and retained snapshots are reviewed.
- [ ] DynamoDB tables are removed if they were only for practice.
- [ ] SNS topics, subscriptions, and SQS queues are cleaned up.
- [ ] CloudWatch alarms and dashboards are reviewed.
- [ ] CloudTrail trails and log storage are retained or removed according to the lab requirement.
- [ ] Lambda functions, triggers, roles, and log groups are reviewed.
- [ ] IAM users, access keys, roles, and policies created for the lab are removed when safe.
- [ ] AWS Billing and Cost Management has been checked.
- [ ] No credentials, private keys, passwords, personal data, or account secrets have been committed to GitHub.

## Suggested GitHub Repository Structure

For a class repository, keep the README concise enough to navigate and place longer code examples in separate files as the material grows:

```text
aws-cloud-practical-labs/
├── README.md
├── ec2/
│   └── user-data-nginx.sh
├── lambda-s3-sns/
│   ├── lambda_function.py
│   └── README.md
├── iam/
│   └── examples/
└── screenshots/
    └── README.md
```

Only commit screenshots that do not expose account IDs, email addresses, public IPs that should remain private, access keys, tokens, or other sensitive information.

---

**Learning tip:** Complete each lab in order, take notes on what you changed, verify the result, and clean up before moving to the next service. For production systems, review AWS Well-Architected guidance, service quotas, security requirements, backups, monitoring, and cost estimates before deploying.
