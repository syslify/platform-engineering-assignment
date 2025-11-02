# Terraform Hands-on Assignment — Build a Mini Cloud Platform

Overview
This assignment guides you step-by-step to build a small, production-like Infrastructure-as-Code (IaC) project with Terraform. It is written for someone who is practically new to Terraform and cloud — you will learn by doing. You will create a modular Terraform project that provisions networking, compute, and a public HTTP endpoint that serves a simple “Hello from Terraform” response. You will also add a CI/CD pipeline to validate and (optionally) plan your Terraform changes on pull requests.

What you will deliver
- A new GitHub repository containing:
  - Terraform code organized into modules/ and root/ (top-level) configuration.
  - README.md (or ASSIGNMENT.md) with clear run instructions.
  - terraform.tfvars.example (no secrets).
  - CI workflow(s) in .github/workflows/ that run linting/format/validate and plan on PRs.
  - Short notes (in README or a separate file) describing issues you hit and what you learned.
- A small evidence artifact: one curl output or screenshot showing the HTTP response from the deployed app.
- Optional: a tiny Dockerfile or app code if you implement a container-based solution.

Estimated time
6–12 hours depending on experience.

Prerequisites
- A computer with a shell/terminal.
- Git and a GitHub account.
- Terraform v1.4+ installed.
- An account with a cloud provider — AWS is recommended for this assignment. You may adapt to another provider, but your README must explain differences.
- (Optional) Docker to test container images locally.

Key learning objectives
- Initialize and apply Terraform projects.
- Structure Terraform projects using modules.
- Configure remote state (best-practice).
- Create and connect basic cloud resources (VPC, subnets, compute, load balancer).
- Securely manage credentials and avoid committing secrets.
- Write reproducible documentation.
- Add a basic CI/CD pipeline to run Terraform checks and produce a plan for review.

Repository layout (suggested)
- README.md (or ASSIGNMENT.md: this document)
- terraform.tfvars.example
- modules/
  - networking/
    - main.tf, variables.tf, outputs.tf
  - compute/
    - main.tf, variables.tf, outputs.tf
  - load_balancer/
    - main.tf, variables.tf, outputs.tf
- root/
  - main.tf, variables.tf, outputs.tf, backend.tf (optional)
- .github/
  - workflows/
    - terraform-ci.yml
- .gitignore

High-level architecture you will build
- VPC with at least two public subnets in different availability zones.
- Security groups for load balancer and compute.
- Compute layer (EC2 instances with user-data that runs a basic HTTP server, or container service such as ECS/Fargate).
- Internet-facing load balancer (ALB/ELB) routing HTTP (port 80) to compute targets.
- Remote Terraform state stored in a remote backend (S3/DynamoDB on AWS) — or a documented explanation if you cannot create it.
- CI/CD pipeline that runs terraform fmt, terraform validate, and terraform plan on pull requests. Optionally post plan output as a PR comment or upload the plan artifact (see hints).

Step-by-step tasks

1) Initialize your repository and branch strategy
- Create a new GitHub repo in your own account (private or public as you prefer).
- Locally, clone it and create a feature branch, e.g., terraform-setup.
- Add a .gitignore that ignores .terraform/, *.tfstate, *.tfstate.* and any credentials.

2) Create a minimal README and plan your approach
- Add a short README with the objective, prerequisites, and how you’ll test.
- Sketch the modules you will implement and the flow of variables/outputs between them.

3) Configure cloud CLI and credentials (do not commit secrets)
- Use the recommended provider CLI to validate credentials (for AWS: aws sts get-caller-identity).
- Use environment variables or the provider’s credential helpers rather than checked-in files.
- Add instructions to README for how to set credentials locally.

4) Scaffold Terraform modules and root config
- Create the modules directory and three modules: networking, compute, load_balancer.
- Each module should contain:
  - main.tf: resources
  - variables.tf: inputs with descriptions and defaults where appropriate
  - outputs.tf: useful outputs for wiring modules
- Create a root/ folder that calls the modules and wires them together.

5) Backend (remote state) setup
- Configure a remote backend so your state is not local:
  - On AWS, create an S3 bucket to hold state and a DynamoDB table for locks. If you can’t create them programmatically due to permissions, create them manually and document the steps.
- Add backend configuration to backend.tf or pass backend configs via CLI during terraform init.
- If remote state is impossible in your environment (e.g., non-cloud or cost-limited), continue with local state but add a README section explaining the risks and how to migrate to remote backend later.

Hints (partial)
- On AWS, backend in S3 uses bucket, key, region and optionally a profile. DynamoDB table is used for state locking.
- backend.tf may contain no secret values — you can parameterize values using CLI -backend-config.

6) Implement the networking module
Requirements:
- Create one VPC with a CIDR block (configurable).
- Create at least two public subnets across different AZs (configurable AZ list or count).
- Create an Internet Gateway and route table(s) to allow public connectivity.
- Create security groups:
  - lb_sg: allows inbound HTTP (80) from 0.0.0.0/0 and outbound to anywhere
  - app_sg: allows inbound HTTP (80) only from lb_sg (security-group reference), and SSH (optional) only from your IP if you need debug access

Design notes and hints:
- Make resource names parameterized by a prefix variable to avoid collisions.
- Return subnet IDs and security group IDs as outputs for other modules.
- Keep module inputs small and purposeful: vpc_cidr, azs/count, public_subnet_cidrs (or auto-generate).
- If the provider supports it, use data sources to select AZs for you.

7) Implement the compute module
Requirements:
- Launch compute instances or a container service that responds to HTTP requests.
- Module inputs should include: subnet_ids, security_group_ids, AMI (or image), instance_type, desired_count.
- Provide user-data (for EC2) or a task definition (for container service) that starts a minimal web server returning a simple static response.

Hints:
- EC2 + user-data: a small bash script that installs a web server and writes an index.html is simple and reliable.
- Example minimal app: Python's http.server, an nginx serving a static index.html, or a one-line busybox httpd.
- If you use container services (ECS/Fargate), your module must create a task definition and service, register it with the ALB target group.
- Add an instance tag or container label that helps you identify resources during testing/cleanup.

8) Implement the load_balancer module
Requirements:
- Create an internet-facing load balancer (ALB or equivalent), listener on port 80, and target group.
- Register compute instances (or the service) as targets.
- Provide health checks (e.g., path / or /health).
- Output the load balancer DNS name or IP.

Hints:
- Use security groups so LB accepts port 80 from the internet and compute accepts port 80 only from LB.
- When registering EC2 instances, use instance IDs. For container services, register the service via target group attachments (IP or instance mode as appropriate).
- Configure health checks to point at the endpoint your user-data serves (e.g., /index.html or /).

9) Wire the modules in the root module
- Call networking to create VPC & subnets.
- Call compute with subnets and app_sg from networking.
- Call load_balancer with LB subnets (public) and LB security group; configure the target group to point to compute outputs.
- Add variable files or a terraform.tfvars.example with example values, but do not include credentials.

10) Add variable documentation and validation
- Add variable descriptions for all inputs.
- Use validation blocks for important inputs (e.g., instance_type not empty, CIDR is a string).
- Include reasonable defaults that are safe (small instance types, small CIDR ranges).

11) Plan and apply carefully
- Run terraform init, terraform plan and review the plan thoroughly.
- Apply to your dev environment. Use small instance types and minimal counts to keep costs low.
- Wait until the LB is marked healthy and then curl the LB DNS name to verify your app is reachable.

12) Test the deployment
- Use curl: curl http://<LB_DNS_NAME>/ and capture the output to include in your submission.
- Verify health check status in the cloud console (optional).
- If you used EC2, you may need to wait for user-data scripts to finish.

13) Clean up
- Destroy the resources with terraform destroy.
- If you created S3 buckets or DynamoDB tables for remote state, decide whether to keep or delete them; document your choice and explain why.
- Ensure that no sensitive state files or credentials remain in the repo.

14) Documentation to include in the repo (README)
Your README must include:
- Clear prerequisites and how to set provider credentials locally (without committing secrets).
- How to create any required backend resources (S3/DynamoDB) and how to initialize the backend in Terraform.
- Step-by-step commands to run:
  - terraform init (with backend config notes)
  - terraform plan -out=tfplan
  - terraform apply tfplan
  - How to retrieve the LB DNS name from outputs
  - terraform destroy
- How to test the running app (curl example).
- Cost-safety notes and cleanup reminders.
- Short “What I learned” section with 3–5 bullet points about non-obvious problems and how you fixed them.

15) Add CI/CD pipeline requirement (new)
Requirement:
- Add a GitHub Actions workflow in .github/workflows/terraform-ci.yml that runs on pull requests and optionally on pushes to main.
- The CI workflow must run at minimum:
  - terraform fmt -check (or run terraform fmt and fail if formatting differs)
  - terraform init (using a working directory, e.g., root/)
  - terraform validate
  - terraform plan -out=tfplan
- The workflow should:
  - Not apply changes automatically.
  - Upload the plan file as a build artifact, or (optionally) convert the plan to human-readable text and add it as a job output or PR comment.
  - Fail the job if fmt/validate/plan fail.
- Security guidance:
  - Do NOT store real cloud credentials in the repository. Use GitHub Actions secrets if you need to access cloud for more advanced checks (e.g., data lookups). For this assignment, you may implement CI that runs only formatting/validation/plan using a read-only or temporary credential set — document your choice in README.
- Optional extra:
  - Add a separate workflow that runs terraform apply in a controlled environment (e.g., a protected branch or when a manual workflow_dispatch is triggered). If you implement this, require approval and document safeguards.

Hints for CI/CD
- Use the official hashicorp/setup-terraform GitHub Action to set Terraform version.
- Use actions/upload-artifact to upload the tfplan file for later download.
- To convert a binary plan to JSON for display, you can run terraform show -json tfplan and save the JSON as an artifact (or parse it with a tool). Be careful including large or sensitive outputs.
- Keep secrets out of public logs. Use masked secrets and avoid echoing credential values.

Deliverable checklist (before pushing)
- [ ] repository has modules/ and root/ Terraform code
- [ ] README.md or ASSIGNMENT.md covers setup, running, testing, CI details, and cleanup
- [ ] terraform.tfvars.example present and documented
- [ ] .github/workflows/terraform-ci.yml present and explained
- [ ] No secrets in the repo (no credentials, no actual tfstate)
- [ ] terraform plan & apply completed in your environment and terraform destroy works
- [ ] Evidence (curl output or screenshot) saved in the repo or linked in the README

Scoring rubric (self-assessment)
- Infrastructure correctness (40%): Modules create required resources; LB routes to compute; security groups configured correctly.
- Terraform best practices (30%): Use of modules, variables, outputs, and (preferably) remote state. Good file organization.
- Documentation & reproducibility (20%): Clear README, tfvars example, CI workflow, and instructions to reproduce/clean up.
- Cost-safety & cleanup (10%): Using small instance sizes; terraform destroy removes resources.

Hints (targeted but NOT full solutions)
- Remote state and locks:
  - Hint: With AWS, S3 stores state and DynamoDB provides locking. You can create the bucket & table manually, or use Terraform itself in a *separate* bootstrap run to create them first.
  - Hint: Don’t commit backend credentials. Use -backend-config CLI arguments to pass sensitive values during terraform init.

- Module boundaries:
  - Hint: Each module should do one job. networking creates networks and security groups; compute creates instances or services; load_balancer creates ALB/ELB and target groups.
  - Hint: Modules communicate via explicit inputs and outputs. Avoid referring to remote modules’ internal names.

- Security Groups:
  - Hint: To allow only the LB to talk to compute, allow inbound to compute_sg from source = aws_security_group.lb_sg.id (provider-specific attribute) rather than 0.0.0.0/0.

- Bootstrapping compute:
  - Hint: A simple bash user-data can install python3 and run `python3 -m http.server 80` serving a small index.html written by the script. Make sure the script is compatible with your cloud's OS (Amazon Linux, Ubuntu, etc.).

- Health checks:
  - Hint: ALB health check path should match the app location (e.g., / or /health). Set healthy threshold low initially to speed tests.

- CI specific:
  - Hint: If you cannot run terraform plan in CI due to missing provider permissions, you can still run terraform fmt and terraform validate. Document that and explain how to enable full plan (what secrets would be needed and how you'd protect them).

Common pitfalls to watch
- Forgetting to add provider credentials or misconfiguring region — terraform may error on plan/apply.
- Security group circular dependencies — pass ids carefully.
- Blocking yourself from deleting resources by creating dependencies on manually created items (take care with state).
- Not waiting enough time after launching instances for the app to be ready. Add a small sleep in user-data if needed.
- CI failing because it cannot access provider data sources — make CI self-contained for format/validate, and document provider access needs separately.

Common troubleshooting steps
- If terraform apply fails due to missing AMI/data lookups: ensure you pass the right filters or use a data source for a public AMI in your region.
- If LB target shows unhealthy: ssh (if enabled) into the instance to verify the server is running and listening on expected port; check security groups and subnet route table configuration.
- If you see a lot of stuck resources on destroy: sometimes resources have implicit dependencies; run terraform plan to see why, and delete specific resources manually only as a last resort (document it).

Stretch (optional)
- Create dev and prod environment variations with separate state backends.
- Add HTTPS termination and redirect HTTP to HTTPS using a managed certificate (ACM on AWS).
- Add autoscaling (ASG) and scaling policies based on CPU or request count.
- Add a CI workflow that runs terraform fmt/validate and plan for PRs (do not apply in CI).
- Add a workflow that runs a trusted manual apply for a staging environment.

Resources and references
- Terraform main docs: https://developer.hashicorp.com/terraform
- Terraform modules: https://developer.hashicorp.com/terraform/language/modules
- AWS VPC: https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html
- AWS ALB & ECS examples: AWS docs and Terraform AWS provider docs
- GitHub Actions (setup-terraform action): https://github.com/hashicorp/setup-terraform
- Actions upload artifact: https://github.com/actions/upload-artifact

Submission instructions
- Push your branch to GitHub and open a pull request to main (or merge to main if you prefer).
- In the PR description include:
  - Short design summary (how modules connect).
  - Commands to run locally to reproduce.
  - One or two screenshots or the curl output (paste text) showing the HTTP response.
  - Notes on anything incomplete or design decisions you intentionally made.
  - Link to the CI run for the PR showing the plan artifact (if implemented).

What to commit first (suggested incremental steps)
- Commit ASSIGNMENT.md (this document).
- Commit .gitignore and terraform.tfvars.example.
- Commit scaffolding for modules with empty files and comments (main.tf, variables.tf, outputs.tf).
- Make small, focused commits: add networking, test; add compute, test; add LB, test; add CI, test.

What I have done for you
- I updated the assignment document to include a CI/CD requirement and detailed guidance for the pipeline. I also added hints about secure handling of Secrets in CI and how to upload plan artifacts for review.

What’s next
- Create the repository and add this ASSIGNMENT.md, a stub .gitignore, and terraform.tfvars.example. Then scaffold the modules directory with the empty main.tf/variables.tf/outputs.tf files for each module and start implementing the networking module first. Commit each small step so you can iterate and test.

If you’d like, I can next provide a starter .gitignore and terraform.tfvars.example file (as content you can paste into the repo), or an example GitHub Actions terraform-ci.yml that implements the minimal CI described above. Which would you like?
