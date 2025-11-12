#I don’t just want to run WordPress. I want to understand how to design WordPress on AWS like a grownup.

##0. Reality check (money + free tier)

You’ve got:

  A. Free tier: 1 small EC2, some S3, some data transfer.
  B. $100 / 6 months: nice, but not infinite.
  C. WordPress likes: PHP, web server, database, shared storage (if multiple nodes), backups, HTTPS.

**So we should start simple and secure, and only add the “fancy autoscaling” once the basics are solid — because the fancier patterns (ALB + Auto Scaling + EFS + RDS/Aurora) start to cost.**


##1. The 3 possible endgames

*A. “Starter, but proper AWS” (cheapest)*
  1× EC2 in a VPC (WordPress + PHP + nginx/Apache)
  RDS MySQL or MariaDB in the same VPC (or even DB on the instance at first)
  ACM cert + ALB (optional) for HTTPS
  SSM for access (no SSH key, no public exposure)
  Backups to S3

**Good for: 1–3 low-traffic sites. Cost: low. Great for learning VPC/IAM/SSM.**

*B. “Real highly available WordPress”*
  ALB in public subnets
  Auto Scaling Group with WordPress AMI (or user data to install WP)
  EFS for /wp-content so all instances share media
  RDS / Aurora Serverless v2 for the database
  AWS WAF attached to ALB
  S3 + CloudFront for offloading media (optional)
  Backups + lifecycle policies

**Good for: “I want to learn how people actually do this in enterprises.”
Cost: higher. You can run it small, but every component adds up.**

*C. “Container nerd” path (for learning modern AWS)*
  ECS on Fargate running a WP container
  EFS for shared content
  Aurora Serverless v2 for DB
  ALB + WAF

**This is very cool, very educational, and very “2025,” but not the cheapest.**

---
***I think we start with A and design it so you can evolve to B without throwing everything away.***
