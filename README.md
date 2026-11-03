
# Three-Tier Web Application on AWS (VPC, ALB, Auto Scaling, RDS)

A highly available, fault-tolerant 3-tier web application built by hand on AWS in the Seoul (`ap-northeast-2`) region across two Availability Zones.

Built as a cloud engineering portfolio project. The build follows the step-by-step guide by trainer Ravindra Bagale: [How to Build a 3-Tier Architecture in AWS Step by Step](https://ravindrabagale.com/aws/build-3-tier-architecture-in-aws-step-by-step-vpc-alb-auto-scaling-rds/).

------------------------------------------------------------------------

## Summary

I built a 3-tier web application on AWS from scratch in the Seoul region. A custom VPC with 8 subnets across 2 Availability Zones separates the public, web, app and database layers. An internet-facing load balancer sends traffic to nginx web servers, which forward PHP requests through an internal load balancer to PHP app servers, which store users in an Amazon RDS MySQL database. Each tier is in private subnets, accepts traffic only from the tier above through chained security groups, and runs in an Auto Scaling group built from tested AMIs. I fixed two real problems along the way: a load balancer in the wrong subnets and a load balancer with the wrong security group.

## 1. What this project does

A user opens a sign-up form in the browser. The request travels through three layers and is saved in a database:

- **Web tier:** nginx serves the HTML form.
- **App tier:** PHP receives the form and hashes the password.
- **Database tier:** Amazon RDS MySQL stores the user.

Each tier runs in private subnets and accepts traffic only from the tier directly above it.

## 2. Architecture

```
Internet
   |  HTTP 80
   v
Internet-facing ALB (public subnets, 2 AZs)
   |
   v
Web tier: nginx servers (private Web subnets) - Auto Scaling Group
   |  only .php requests
   v
Internal ALB (private App subnets)
   |
   v
App tier: PHP + php-fpm servers (private App subnets) - Auto Scaling Group
   |  MySQL 3306
   v
Amazon RDS MySQL (private DB subnets)
```

### Network plan (VPC `10.0.0.0/16`)

| Subnet        | AZ              | CIDR          | Type    |
|---------------|-----------------|---------------|---------|
| PublicSubnet1 | ap-northeast-2a | 10.0.0.0/20   | Public  |
| PublicSubnet2 | ap-northeast-2b | 10.0.16.0/20  | Public  |
| WebSubnet1    | ap-northeast-2a | 10.0.32.0/20  | Private |
| WebSubnet2    | ap-northeast-2b | 10.0.48.0/20  | Private |
| AppSubnet1    | ap-northeast-2a | 10.0.64.0/20  | Private |
| AppSubnet2    | ap-northeast-2b | 10.0.80.0/20  | Private |
| DBSubnet1     | ap-northeast-2a | 10.0.96.0/20  | Private |
| DBSubnet2     | ap-northeast-2b | 10.0.112.0/20 | Private |

### Security group chain (each tier trusts only the tier above)

| Security group | Attached to                           | Allows inbound            |
|----------------|---------------------------------------|---------------------------|
| WebAlbSG       | Internet-facing ALB                   | HTTP 80 from the internet |
| WebEc2SG       | nginx servers                         | HTTP 80 from WebAlbSG     |
| AppAlbSG       | Internal ALB                          | HTTP 80 from WebEc2SG     |
| AppEc2SG       | PHP servers                           | HTTP 80 from AppAlbSG     |
| DbSG           | RDS MySQL                             | MySQL 3306 from AppEc2SG  |
| JumpSG / DevSG | JumpServer / dev servers (setup only) | SSH from my IP only       |

## 3. AWS services and tools used

- **Networking:** VPC, subnets, Internet Gateway, route tables, security groups
- **Compute:** EC2 (Amazon Linux 2023, t3.micro), AMIs, Launch Templates
- **Scaling and load balancing:** Auto Scaling Groups, Application Load Balancers (one internet-facing, one internal)
- **Database:** Amazon RDS for MySQL, DB subnet group
- **Software:** nginx, PHP, php-fpm, MySQL client
- **Access:** SSH with a key pair, JumpServer for one-time database setup

## 4. High availability and fault tolerance

- **Two Availability Zones:** every tier has servers in both AZs, so one AZ failing does not take a tier down.
- **Auto Scaling:** desired 2, minimum 1, maximum 4 servers per tier. Target tracking on average CPU keeps the load balanced.
- **Health checks:** the load balancers check each server, and Auto Scaling replaces unhealthy servers automatically.
- **Immutable servers:** servers are built from tested AMIs (`webami`, `phpami`) and are never changed by hand.
- **Private tiers:** web, app and database servers have no public IP and are not reachable from the internet directly.

## 5. How I built it (build order)

The build goes bottom to top because each tier needs an address from the tier below. The security rules read top to bottom.

1.  Create the VPC and 8 subnets
2.  Create the Internet Gateway and route tables
3.  Create all security groups, then add the chained rules
4.  Create the DB subnet group and RDS MySQL
5.  Launch a JumpServer and create the `insta` database and `users` table
6.  Build the PHP development server, test it, create `phpami`
7.  Create the app-tier Launch Template, Auto Scaling Group and internal ALB
8.  Build the nginx development server with the PHP proxy, create `webami`
9.  Create the web-tier Launch Template, Auto Scaling Group and internet-facing ALB
10. Test the full flow through the ALB DNS name

## 6. Build steps and results

These are the steps I completed in the AWS Console and the result of each one.

### Network

| # | Step | What I did |
|---|---|---|
| 1 | VPC created | I created the VPC `ThreeTierVpc` with CIDR `10.0.0.0/16` in the Seoul region and enabled DNS resolution. It shows as Available. |
| 2 | Eight subnets in two AZs | I created 8 subnets across `ap-northeast-2a` and `2b`: 2 public, 2 web, 2 app and 2 DB, each `/20` from `10.0.0.0/20` to `10.0.112.0/20`. |
| 3 | VPC resource map | I opened the resource map to check that all 8 subnets, the route tables and the Internet Gateway are connected to `ThreeTierVpc`. |
| 4 | Public route to Internet Gateway | I created `ThreeTierPublicRouteTable`, added a `0.0.0.0/0` route to the Internet Gateway and attached it to the two public subnets. The other subnets keep only the local route, so they stay private. |

### Security

| # | Step | What I did |
|---|---|---|
| 5 | All security groups | I created one security group per layer: WebAlbSG, WebEc2SG, AppAlbSG, AppEc2SG and DbSG, plus JumpSG and DevSG for setup only. |
| 6 | Chained inbound rule 1 | I edited `WebEc2SG` to allow HTTP 80 only from the internet-facing ALB's security group (WebAlbSG). |
| 7 | Chained inbound rule 2 | I edited `AppAlbSG` (internal ALB) to allow HTTP 80 only from `WebEc2SG`, so only the web servers can reach the app tier. |
| 8 | Chained inbound rule 3 | I edited `AppEc2SG` (PHP servers) to allow HTTP 80 only from `AppAlbSG`. |
| 9 | Chained inbound rule 4 | I edited `DbSG` to allow MySQL 3306 from `AppEc2SG`. A second 3306 rule from my setup security group let the JumpServer create the database. |

### Database

| # | Step | What I did |
|---|---|---|
| 10 | DB subnet group | I created `threetierdbsubnetgroup` with the two private DB subnets, so RDS is placed only in private subnets. Its status is Complete. |
| 11 | RDS available, public access = No | I launched the MySQL instance `database-1` (`db.t4g.micro`, Single-AZ) in the DB subnet group with public access set to No and `DbSG` attached. It shows as Available. |
| 12 | JumpServer running | I launched a temporary `Jumpserver` EC2 instance in `PublicSubnet1` (Amazon Linux 2023) to reach the private database, with SSH allowed only from my IP. |
| 13 | MySQL connected, users table | From the JumpServer I connected to RDS, created the `insta` database and the `users` table (`id`, `name`, `mobile`, `password`), and checked the saved rows with `select * from users`. Passwords are stored as bcrypt hashes. |

### AMIs, Auto Scaling and Load Balancers

| # | Step | What I did |
|---|---|---|
| 14 | AMIs | After testing the development servers, I created the AMIs `phpami` for the app tier and `Webami` / `WebTier2` for the web tier. Auto Scaling builds new servers from these images. |
| 15 | App and Web Auto Scaling Groups | I created `AppTierASG` and `WebTierASG` from launch templates built on the AMIs, with a maximum of 4 servers each. Both show At desired capacity with healthy instances. |
| 16 | Internal ALB | I created `AppTierInternalALB` as an internal load balancer in the private subnets across 2 AZs and attached `AppAlbSG` to it. |
| 17 | App target group healthy | I checked `AppTierTG` and confirmed that the `AppTierServer` instance behind the internal ALB shows Healthy. |
| 18 | Internet-facing ALB | I created `WebTierPublicALB` as an internet-facing load balancer in the two public subnets, with `WebAlbSG` attached. Both load balancers are Active. |
| 19 | Internet-facing ALB details | I checked that the ALB uses one public subnet in each AZ (`2a` and `2b`) and copied its DNS name to open the site in the browser. |
| 20 | Web target group healthy | I checked `WebTierTG` and confirmed that the nginx server behind the internet-facing ALB shows 1 healthy and 0 unhealthy. |

## 7. Problems I faced and what I learned

### Problem 1: My internet-facing load balancer was placed in the private Web subnets

**Problem:** I created the internet-facing Application Load Balancer from the Auto Scaling group wizard and selected the Web subnets (private) instead of the Public subnets. The load balancer was created and showed as Active, but the web page could not be opened from my browser using its DNS name.

**Why it happened:** A subnet is public only if its route table has a route (`0.0.0.0/0`) to an Internet Gateway. The Web subnets only have the local route, so a load balancer placed there has no path to or from the internet, even though its scheme says "internet-facing". The scheme decides the type of address the load balancer gets, but the subnet's route table decides whether that address is actually reachable.

**How I found it:** I checked the load balancer's Network mapping and then the route table of each subnet it used. The subnets had no route to the Internet Gateway, while PublicSubnet1 and PublicSubnet2 did.

**How I fixed it:** I changed the load balancer's subnets to PublicSubnet1 and PublicSubnet2 (one per AZ), and then the site opened through the ALB DNS name.

**What I learned:**

- "Public" and "private" come from the route table, not from the name of a subnet or the load balancer scheme.
- An internet-facing load balancer must sit in public subnets in at least two AZs, while the servers behind it can stay private.
- When something is unreachable, check in order: subnet and route table, then security groups, then the app itself.
- Build the infrastructure bottom to top, but write the security rules top to bottom.
- A security group should use another security group as its source, not an IP address, because Auto Scaling servers get new IPs all the time.
- Never change a live server. Change a development server, create a new AMI and let Auto Scaling replace the old servers.

### Problem 2: The load balancer had the wrong security group

**Problem:** Targets showed Unhealthy and the page returned an error.

**Why it happened:** The Auto Scaling wizard attached the launch template's security group to the new load balancer. My security group chain only allows each tier to receive traffic from the load balancer's own group, so the traffic was blocked.

**How I fixed it:** I opened the load balancer's Security tab, removed the wrong group and attached `WebAlbSG` (and `AppAlbSG` for the internal one).

**What I learned:**

- Why each tier needs its own security group, with another security group as the source instead of an IP address
- Why servers are built from AMIs and never changed by hand
- How health checks and Auto Scaling replace broken servers on their own
- Why the database sits in private subnets with no public access

## 8. Limitations and future improvements

Being honest about the lab version:

- RDS is Single-AZ in this lab to stay in the free tier. Production should use Multi-AZ for automatic failover.
- The database password is stored in the PHP code in this lab. Production should use AWS Secrets Manager.
- HTTP only. Production should add HTTPS with ACM and a custom domain.
- No NAT Gateway. Servers get their software from AMIs, so private servers do not need internet access.
- **Next steps:** Terraform for the whole stack, CloudWatch alarms with SNS, and a CI/CD pipeline.

## 9. Clean-up

All resources were deleted after testing to avoid charges: Auto Scaling groups, load balancers, target groups, RDS, AMIs and snapshots, JumpServer, then the VPC.

## 10. Author

**Rohit Patil** - Cloud Engineering Portfolio Project

- GitHub:(https://github.com/rohitpatil019)
