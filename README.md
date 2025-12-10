# terraform-atlantis-pipeline-demo

**How to deploy a Terraform-based infrastructure production GitLab pipeline with Atlantis in your home DevOps lab?**

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/ad90049e-abf5-4233-bb62-c2ca22ff9fdc" />

In this article, I will demonstrate how to replicate a Terraform-based infrastructure production pipeline with Atlantis directly in your home DevOps lab. Please follow the instructions below:

1. Launch an Ubuntu 24.04 LTS-based VM (AWS EC2 or on any cloud platform; I used AWS EC2)
   
Region: us-east-1

Instance: t2.micro

Security Group:

Allow inbound:

22 (your public IP)

4141 (GitLab → Atlantis webhook)

Also, could you attach an IAM Role to the VM, with enough permissions to create Terraform-based resources?

2. SSH to the VM and Install Docker:

sudo apt update

sudo apt install -y git

sudo apt install -y docker.io

sudo systemctl enable — now docker # It is a double dash, not a hyphen before now

3. On your browser, navigate to: https://gitlab.com/users/sign_up

Now, sign up with your details or use: Continue with google

4. After you have signed up, log in to your GitLab account and navigate to: https://gitlab.com/bhavukm/atlantis

5. Now, fork this repository. This repo contains the Atlantis pipeline logic.

6. Then, go to: Settings >> Access Tokens >> Add new token:

Choose the options as in the screenshot below, create a new access token, and save it:

<img width="813" height="752" alt="image" src="https://github.com/user-attachments/assets/5b597757-0654-4ca0-830a-a2ccdfff5b60" />

Also, create an IAM User in AWS and generate its access keys (not recommended for real-world production. Using here for simplicity)

Please ensure the IAM User has sufficient permissions to create the resources you mention in your Terraform script files. To keep it simple, just add the Administrator IAM policy to the IAM user.

In production: OIDC / Workload Identity Federation (recommended for SaaS GitLab)

Now add the IAM User access keys in GitLab (settings >> CI/CD >> variables)as shown below:

<img width="577" height="623" alt="image" src="https://github.com/user-attachments/assets/dc1bf1bf-e789-4351-aa0a-06586ff9de1d" />

<img width="828" height="435" alt="image" src="https://github.com/user-attachments/assets/e9fd1972-98b0-43ad-bffc-9975b6a26e74" />

7. Now, go back to your EC2 (VM) terminal, clone the forked repo:

For https: git clone -b fix/atlantis-yaml https://gitlab.com/<GitLab-username>/<repo>.git

For SSH (configure with ssh-keygen command, copy the SSH public key to GitLab for authentication): git clone -b fix/atlantis-yaml git@gitlab.com:<GitLab-username>/<repo>.git

cd <repo-folder>/application/environments/dev

Example path for root user: /root/atlantis/atlantis/atlantis/application/environments/dev

create a new git branch: git checkout -b atlantis-test

Create your Terraform files here(.tf, .tfvars, etc):

<img width="771" height="183" alt="image" src="https://github.com/user-attachments/assets/cc7779e0-8f21-43c4-9074-f33ca37c36a4" />

8. Now, run the Atlantis Instance via a Docker Container:

docker run -d — name atlantis -p 4141:4141 -e ATLANTIS_GITLAB_USER=<GitLab-Username> -e ATLANTIS_GITLAB_TOKEN=<GitLab-Access-Token> -e ATLANTIS_REPO_ALLOWLIST=gitlab.com/<GitLab-Username>/<repo> -e ATLANTIS_LOG_LEVEL=info runatlantis/atlantis:latest server

This will create a Docker container for the Atlantis Instance running on your VM. Verify: sudo docker ps

9. Now, log in to this Docker container to update the Terraform version. This is to resolve an issue with the Terraform version that I faced. We need it at least at 1.3.0 here.

docker exec -it atlantis sh

terraform version

wget https://releases.hashicorp.com/terraform/1.3.0/terraform_1.3.0_linux_amd64.zip

apk add unzip

unzip terraform_1.3.0_linux_amd64.zip

mv terraform /usr/local/bin/

terraform version

“Ctrl + D” to log out of the Docker container

10. Next, we need to configure the GitLab webhook for the Atlantis instance. Navigate to your GitLab project:

Go to: Project → Settings → Webhooks

URL: http://<VM-Public-IP>:4141/events

<img width="812" height="431" alt="image" src="https://github.com/user-attachments/assets/690c7a02-0413-4e3c-93ab-2cd2ab4a5a09" />

<img width="475" height="613" alt="image" src="https://github.com/user-attachments/assets/ab6fe5ac-0487-4d46-869d-68468b7a7256" />

Create the webhook and test it. If successful, you will see: Status code 200

11. Back to your terminal, commit and push your changes to your GitLab repo:

git add <.tf files>

git commit -m “Atlantis-based Terraform Deployment”

git push origin <branch-name>

12. On your GitLab, create the MR (Merge Request):

    <img width="828" height="221" alt="image" src="https://github.com/user-attachments/assets/716b0931-15a9-426e-9f80-2ee563798551" />

    <img width="811" height="690" alt="image" src="https://github.com/user-attachments/assets/7c14b401-6198-460f-ae19-27d91fe21f0a" />

    If the MR pipeline succeeds, check the comment section to see “atlantis plan” details:

    <img width="828" height="478" alt="image" src="https://github.com/user-attachments/assets/2274a535-5574-4945-97b6-e9affa9b5463" />

    <img width="720" height="287" alt="image" src="https://github.com/user-attachments/assets/cf7edd8e-e6a4-4bd7-a0bc-6737792dca41" />

    13. If you are satisfied with the plan, to apply, just comment in the MR:

atlantis apply

This will apply your “atlantis plan”. If this succeeds, you will see this type of output in the MR comment section:

<img width="828" height="293" alt="image" src="https://github.com/user-attachments/assets/e3ead4d9-d4f4-4b13-809e-07d5339d7c55" />

14. Verify the resources in your AWS Management console:

    <img width="828" height="488" alt="image" src="https://github.com/user-attachments/assets/ad439dd7-31cc-4c16-baf5-c10a55728450" />

    15. To destroy resources, comment in the MR:

To check the destroy plan: atlantis plan -d application/environments/dev -w default -- -destroy # After default, it is space, then double dash, then space, then single dash, destroy

<img width="572" height="77" alt="image" src="https://github.com/user-attachments/assets/d20ec305-3fb5-4dc3-9998-d09506588c8d" />

To apply the destroy plan: atlantis apply

16. After successful resource destruction, you can merge the feature branch to the main branch:

    <img width="658" height="598" alt="image" src="https://github.com/user-attachments/assets/0e3a6d43-9020-4433-9836-60447d230359" />

    <img width="362" height="167" alt="image" src="https://github.com/user-attachments/assets/b0c3b8ee-f491-4625-9f57-0c71e48bfac3" />

    That is it. Congratulations! You now have a Terraform-based Atlantis pipeline to provision your resources.















