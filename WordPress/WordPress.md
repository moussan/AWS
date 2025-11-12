# I don’t just want to run WordPress. I want to understand how to design WordPress on AWS like a grownup.

### 0. Reality check (money + free tier)

You’ve got:

- Free tier: 1 small EC2, some S3, some data transfer.
- $100 / 6 months: nice, but not infinite.
- WordPress likes: PHP, web server, database, shared storage (if multiple nodes), backups, HTTPS.

**So we should start simple and secure, and only add the “fancy autoscaling” once the basics are solid — because the fancier patterns (ALB + Auto Scaling + EFS + RDS/Aurora) start to cost.**


### 1. The 3 possible endgames

A. “Starter, but proper AWS” (cheapest)
- 1× EC2 in a VPC (WordPress + PHP + nginx/Apache)
- RDS MySQL or MariaDB in the same VPC (or even DB on the instance at first)
- ACM cert + ALB (optional) for HTTPS
- SSM for access (no SSH key, no public exposure)
- Backups to S3

**Good for: 1–3 low-traffic sites. Cost: low. Great for learning VPC/IAM/SSM.**

B. “Real highly available WordPress”
- ALB in public subnets
- Auto Scaling Group with WordPress AMI (or user data to install WP)
- EFS for /wp-content so all instances share media
- RDS / Aurora Serverless v2 for the database
- AWS WAF attached to ALB
- S3 + CloudFront for offloading media (optional)
- Backups + lifecycle policies

**Good for: “I want to learn how people actually do this in enterprises.”
Cost: higher. You can run it small, but every component adds up.**

C. “Container nerd” path (for learning modern AWS)
- ECS on Fargate running a WP container
- EFS for shared content
- Aurora Serverless v2 for DB
- ALB + WAF

**This is very cool, very educational, and very “2025,” but not the cheapest.**


I think we start with A and design it so you can evolve to B without throwing everything away.
---

### 2. Phase-by-phase plan

Phase 1 – Secure the account (15–30 min)
This is boring but absolutely part of “architecting in AWS.”
- Turn on MFA on the root user.
- Create an admin IAM user/role and stop using root.
- Create a Budget (e.g. $20/month alert).
- Turn on AWS CloudTrail and Security Hub (even in basic form).

Now you’re behaving like someone who manages real infra.

Phase 2 – Networking skeleton
Create a proper VPC so we don’t have to redo it later:
- 1 VPC (10.0.0.0/16)
- 2 public subnets (for ALB / bastion if ever needed)
- 2 private subnets (for EC2/DB)
- 1 NAT Gateway (optional at first — to save $$ you can start with public EC2 and lock it down)

If you want to save money right now: put the WordPress EC2 in a public subnet with a security group that only allows 80/443 from the world and SSM for management. Later we move it private.

Phase 3 – First WordPress EC2
Goal: get a WP up, but built “the AWS way.”

- Launch a t2.micro / t3.micro (free-tier eligible) Amazon Linux 2023 or Ubuntu.
- Attach a security group: inbound 80/443, and SSM.
- Use Session Manager (SSM) to get in — no SSH port open.
- Install: nginx/Apache, PHP-FPM, and WordPress.

For the DB, you have 2 choices right now:

+ Same EC2 (cheapest, simplest, least HA) — good for day 1. OR
+ RDS MySQL (better practice) — small instance, auto-backups.

Since you said “resilient” and “learning,” I’d say: let’s put the DB on RDS early, so later when we add more WP instances we don’t have to migrate the DB.

Phase 4 – HTTPS + domain + WAF entry point
You said “security focused” and “firewall or WAF.”

- Put an Application Load Balancer (ALB) in front of your EC2.
- Get a cert from AWS Certificate Manager (ACM) for your domain (free).
- Attach AWS WAF to the ALB (start with managed rules).
- Point your domain in Route 53 (or your registrar) to the ALB.

Now you have:
Internet → ALB (HTTPS, WAF) → EC2 (WordPress) → RDS (DB)

That’s already a solid, modern pattern.

Phase 5 – Make it multi-site
WordPress can host multiple sites from the same install (multisite) or you can run multiple virtual hosts on the same EC2. Easiest:

- Keep one EC2 but host multiple WP sites in different directories / vhosts.
- ALB rules can route different hostnames to the same target group.

Later, if you want strict isolation, you spin up separate EC2s in the same pattern.

Phase 6 – Make it scale (the fun part)
Right now everything is on 1 EC2. To scale horizontally we need to make WP stateless:

- Bake or script install so new EC2s can come up ready.
- Move /wp-content off the instance:
+ Easiest AWS way: mount EFS on each instance and point WordPress uploads there.
+ Or use S3 offload plugin (a bit more WP-y).
- Put the EC2s in an Auto Scaling Group behind the ALB.
- Add target tracking scaling policy (scale out when CPU > 50%).

Now when traffic spikes, new WP instances come up, mount EFS, connect to the same RDS, and serve traffic.

That’s the “proper” WordPress on AWS.
