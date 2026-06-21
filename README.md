# MuchToDo Backend Deployment

This is my Month 1 assessment for the Tinyuka 2025 DevOps program. The task was to provision AWS infrastructure with Terraform and containerize a Golang backend app (MuchToDo) using Docker, then deploy it to EC2 behind a load balancer.

## What you need before starting

- AWS CLI set up with an IAM user that has the right permissions  
- Terraform installed (I used 1.5+)  
- Docker and Docker Compose  
- An SSH key pair for connecting to EC2  
- Some kind of Linux environment with at least 2GB RAM for building the Docker image (more on why below)

## What gets created

Terraform sets up a VPC with public and private subnets across two availability zones, a NAT Gateway so the private instances can reach the internet, and three EC2 instances \- a bastion host you SSH into first, the backend server where the app actually runs, and a MongoDB server. There's also an Application Load Balancer in front of everything, and security groups locking down which instance can talk to which.

## Step 1: Spin up the infrastructure

Go into the terraform folder and copy the example vars file:

cd terraform

cp terraform.tfvars.example terraform.tfvars

Open terraform.tfvars and put in your own IP address (run `curl https://checkip.amazonaws.com` to get it) in the my\_ip variable, formatted like 203.0.113.5/32. This matters because your IP changes depending on your network, so you'll need to update this every time you come back to work on it.

Then:

terraform init

terraform plan

terraform apply

Once it's done, write down the outputs \- you'll need the alb\_dns\_name, bastion\_public\_ip and backend\_private\_ip for the next steps.

## Step 2: Build the Docker image

Quick note on why this step looks the way it does. The backend EC2 instance is a t3.micro, which only has 1GB of RAM. Trying to build the Go binary directly on that instance kept failing \- the Go compiler would either get killed for using too much memory or take close to an hour even with swap space added. So instead I build the image somewhere with more memory (I used a Vagrant VM) and just move the finished image over to EC2.

docker build \-t muchotodo-backend .

docker save muchotodo-backend \-o muchotodo-backend.tar

That tar file is what gets copied to the server later.

## Step 3: Get onto the backend server and set it up

The backend server sits in a private subnet, so you can't SSH to it directly \- you have to go through the bastion host first. This command does both hops in one go:

ssh \-o ProxyCommand="ssh \-i your-key \-W %h:%p ec2-user@BASTION\_IP" \-i your-key ec2-user@BACKEND\_PRIVATE\_IP

If this is a fresh instance (first time connecting after terraform apply), it won't have Docker installed yet:

sudo yum install docker git \-y

sudo systemctl start docker

sudo systemctl enable docker

sudo usermod \-aG docker ec2-user

Install docker-compose too since it doesn't come with Amazon Linux:

sudo curl \-L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname \-s)-$(uname \-m)" \-o /usr/local/bin/docker-compose

sudo chmod \+x /usr/local/bin/docker-compose

Also add some swap space \- even just running the containers (not building) is tight on 1GB RAM:

sudo fallocate \-l 2G /swapfile

sudo chmod 600 /swapfile

sudo mkswap /swapfile

sudo swapon /swapfile

After adding yourself to the docker group you need to disconnect and reconnect for it to take effect.

Now clone the repo onto the server:

git clone https://github.com/Reallife-1/backend-deployment-assessment.git

cd backend-deployment-assessment

You'll need to create a .env file here too since it's gitignored and won't come with the clone:

PORT=8080

MONGO\_URI=mongodb://mongodb:27017/much\_todo\_db

DB\_NAME=much\_todo\_db

JWT\_SECRET\_KEY=change-this-secret

JWT\_EXPIRATION\_HOURS=72

LOG\_LEVEL=INFO

LOG\_FORMAT=json

## Step 4: Move the image over and start everything

Back on your own machine (open a new terminal, keep the SSH session running), send the tar file through the bastion the same way:

scp \-o ProxyCommand="ssh \-i your-key \-W %h:%p ec2-user@BASTION\_IP" \-i your-key muchotodo-backend.tar ec2-user@BACKEND\_PRIVATE\_IP:/home/ec2-user/

This takes a few minutes depending on your connection. Back on the EC2 session:

docker load \-i muchotodo-backend.tar

docker-compose up \-d

docker-compose ps

Both containers (backend and mongodb) should show as Up.

## Step 5: Check it's actually working

On the EC2 instance:

curl http://localhost:8080/health

From your own machine, hitting it through the load balancer:

curl http://YOUR\_ALB\_DNS\_NAME/health

Both should give back something like {"cache":"disabled","database":"ok"}.

## Cleaning up

The NAT Gateway costs money by the hour, so don't leave this running. When you're done for the session:

cd terraform

terraform destroy

Next time you come back, check your IP again before running apply, since it probably changed:

curl https://checkip.amazonaws.com

## A few things worth knowing

terraform.tfvars has your real IP in it and is gitignored on purpose \- never commit it. terraform.tfstate on the other hand is committed since that was a requirement for this assignment. The muchotodo-backend.tar file is also gitignored, it's just a local build artifact and isn't meant to live in the repo. Same with .env \- you'll have to recreate it by hand any time you spin up a fresh instance.  
