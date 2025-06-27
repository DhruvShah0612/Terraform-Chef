# 🧪 Chef + Terraform AWS Automation Tasks

This repository demonstrates how to automate infrastructure and configuration management on AWS EC2 using **Terraform** and **Chef**. Each task is step-by-step and includes scripts, Terraform code, and verification instructions.

---

## 🧰 Task 1: Install Chef Workstation on Local Machine (Windows)

### 📌 Objective
Install [Chef Workstation](https://community.chef.io/downloads/tools/workstation?os=ubuntu) on your **local Windows machine** to use tools like `chef`, `knife`, `chef-solo`, `chef-client`, etc., from your terminal.

#### 🔧 Step 1: Download the Official Chef Workstation Installer

1. Visit the official download page:  
   👉 https://community.chef.io/downloads/tools/workstation?os=ubuntu

2. Select:
   - **Platform:** Windows
   - **Architecture:** x86_64

3. Click **Download** to get the `.msi` installer file.

---

#### 🔧 Step 2: Install Chef Workstation

1. **Double-click** the downloaded `.msi` file.
2. Follow the installer steps:
   - Accept the license
   - Use the default installation directory
   - Click **Install**
3. Once complete, click **Finish**

---

#### 🔄 Step 3: Restart Terminal

- **Close and reopen** PowerShell or Git Bash.

---

#### 🔍 Step 4: Verify the Installation

Open **PowerShell** and run:

```powershell
chef -v
```
You should see output like:
```
Chef Workstation version: 23.x.x
Chef Infra Client version: 18.x.x
```
--- 

## 🛠️ Task 2: Launch EC2 & Install Apache via Chef Using Terraform
### 🔧 Step 1: Create `install_chef_apache.sh`
```bash
#!/bin/bash -xe

# Update and install required tools
apt update -y
apt install -y curl git unzip

# Install Chef Infra Client
curl -L https://omnitruck.chef.io/install.sh | bash

# Create cookbook and recipe directory
mkdir -p /root/cookbooks/apache_install/recipes

# Write the Apache install recipe
cat <<EOF > /root/cookbooks/apache_install/recipes/install_apache.rb
package 'apache2' do
  action :install
end

service 'apache2' do
  action [:enable, :start]
end
EOF

# Write metadata.rb
cat <<EOF > /root/cookbooks/apache_install/metadata.rb
name 'apache_install'
maintainer 'Your Name'
maintainer_email 'you@example.com'
license 'All Rights Reserved'
description 'Installs Apache Web Server'
version '0.1.0'
EOF

# Write Chef Solo config
cat <<EOF > /root/solo.rb
cookbook_path ['/root/cookbooks']
EOF

# Run Chef recipe with license accepted
chef-client -z -c /root/solo.rb -o 'apache_install::install_apache' --chef-license accept
```

### 📦 Step 2: update `main.tf`
```hcl
provider "aws" {
  region  = "ap-south-1"
  profile = "default" # Uses AWS CLI credentials
}

resource "aws_key_pair" "deployer" {
  key_name   = "terraform-key"
  public_key = file("~/.ssh/id_rsa.pub") # Update path if needed
}

resource "aws_security_group" "apache_sg" {
  name        = "apache-sg"
  description = "Allow HTTP and SSH"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "apache_server" {
  ami                         = "ami-0f58b397bc5c1f2e8" # Ubuntu 20.04 (ap-south-1)
  instance_type               = "t2.micro"
  key_name                    = aws_key_pair.deployer.key_name
  vpc_security_group_ids      = [aws_security_group.apache_sg.id]
  associate_public_ip_address = true

  user_data = file("install_chef_apache.sh")

  tags = {
    Name = "Chef-Apache-Server"
  }
}
```

### ▶️ Step 3: Deploy Using Terraform
```bash
terraform init
terraform apply -auto-approve
```

### 🔍 Step 4: Verify Apache Web Server

1. Go to AWS EC2 console → find your instance.
2. Copy the Public IP.
3. Visit in browser:
   
```
http://<EC2_PUBLIC_IP>
```
✅ You should see the Apache2 Ubuntu Default Page.

### 🧪 Manual Verification
```bash
ssh -i ~/.ssh/id_rsa ubuntu@<EC2_PUBLIC_IP>
sudo -i
cat /var/log/user-data.log
systemctl status apache2
ls /root/cookbooks/apache_install/
ls /root/cookbooks/apache_install/recipes/
cat /root/cookbooks/apache_install/recipes/install_apache.rb
```
---

## 👤 Task 3: Write a Recipe to Create a User and Set Password Using Chef + Terraform
- Create a Linux user (`devuser`)
- Set a secure password

###  step1: 📄 create file `install_chef_user.sh`
```bash
#!/bin/bash -xe

# Install base tools
apt update -y
apt install -y curl git unzip openssl

# Install Chef Infra Client
curl -L https://omnitruck.chef.io/install.sh | bash

# Create Chef cookbook for user management
mkdir -p /root/cookbooks/user_setup/recipes

# Generate hashed password (example: "admin@123")
HASHED_PASS=$(openssl passwd -6 "admin@123")

# Write the user creation recipe
cat <<EOF > /root/cookbooks/user_setup/recipes/create_user.rb
user 'devuser' do
  comment 'DevOps User'
  home '/home/devuser'
  shell '/bin/bash'
  password '${HASHED_PASS}'
  manage_home true
  action :create
end
EOF

# Metadata for the cookbook
cat <<EOF > /root/cookbooks/user_setup/metadata.rb
name 'user_setup'
maintainer 'Your Name'
license 'All Rights Reserved'
description 'Creates a user with Chef'
version '0.1.0'
EOF

# Solo configuration file
cat <<EOF > /root/solo.rb
cookbook_path ['/root/cookbooks']
EOF

# Run the Chef recipe
chef-client -z -c /root/solo.rb -o 'user_setup::create_user' --chef-license accept
```

### 📦 Step 2: add this `main.tf`
```hcl
resource "aws_instance" "user_server" {
  ami                         = "ami-0f58b397bc5c1f2e8" # Ubuntu 20.04
  instance_type               = "t2.micro"
  key_name                    = aws_key_pair.deployer.key_name
  vpc_security_group_ids      = [aws_security_group.apache_sg.id]
  associate_public_ip_address = true

  user_data = file("install_chef_user.sh")

  tags = {
    Name = "Chef-User-Server"
  }
}
```
Apply with:
```bash
terraform apply -auto-approve
```

### 🧪 Verify on EC2
```bash
ssh -i ~/.ssh/id_rsa ubuntu@<EC2_PUBLIC_IP>
```

Check user and password:
```bash
# Confirm user exists
getent passwd devuser

# View password hash
sudo cat /etc/shadow | grep devuser

# Switch to new user
sudo su - devuser
```

---

## 🔧 Task 4: Use a Cookbook to Manage a Package and Service (`httpd`)
### 📄 step1: create file `install_httpd.sh`
```bash
#!/bin/bash -xe

# Update system and install dependencies
apt update -y
apt install -y curl git unzip

# Install Chef Infra Client
curl -L https://omnitruck.chef.io/install.sh | bash

# Create cookbook directory
mkdir -p /root/cookbooks/httpd_manage/recipes

# Recipe: install and manage apache2
cat <<EOF > /root/cookbooks/httpd_manage/recipes/manage_httpd.rb
package 'apache2' do
  action :install
end

service 'apache2' do
  action [:enable, :start]
end
EOF

# Metadata
cat <<EOF > /root/cookbooks/httpd_manage/metadata.rb
name 'httpd_manage'
maintainer 'Your Name'
license 'All Rights Reserved'
description 'Manages apache2 package and service'
version '0.1.0'
EOF

# Chef Solo Config
cat <<EOF > /root/solo.rb
cookbook_path ['/root/cookbooks']
EOF

# Run the recipe
chef-client -z -c /root/solo.rb -o 'httpd_manage::manage_httpd' --chef-license accept
```

### 📦 Step 2: add this `main.tf`
```hcl
resource "aws_instance" "httpd_server" {
  ami                         = "ami-0f58b397bc5c1f2e8" # Ubuntu 20.04
  instance_type               = "t2.micro"
  key_name                    = aws_key_pair.deployer.key_name
  vpc_security_group_ids      = [aws_security_group.apache_sg.id]
  associate_public_ip_address = true

  user_data = file("install_httpd.sh")

  tags = {
    Name = "Chef-HTTPD-Server"
  }
}
```
Apply with:
```
terraform apply -auto-approve
```

### 🔍 Verify Apache

1. Go to AWS EC2 console → find your instance.
2. Copy the Public IP.
3. Visit in browser:
   
```
http://<EC2_PUBLIC_IP>
```

✅ You should see the Apache2 Ubuntu Default Page.

---

## 🧪 Task 5: Run a Recipe Using chef-client in Local Mode (`--local-mode` / `-z`)
### 📄 Step 1: Create `install_local_mode.sh`
```bash
#!/bin/bash -xe

# Update and install required packages
apt update -y
apt install -y curl unzip

# Install Chef Infra Client
curl -L https://omnitruck.chef.io/install.sh | bash

# Create cookbook directory and recipe
mkdir -p /root/cookbooks/local_mode/recipes

# Recipe: Create a simple file
cat <<EOF > /root/cookbooks/local_mode/recipes/hello_file.rb
file '/root/hello_from_chef.txt' do
  content 'Hello, Chef is working in local mode!'
  owner 'root'
  group 'root'
  mode '0644'
  action :create
end
EOF

# Metadata for the cookbook
cat <<EOF > /root/cookbooks/local_mode/metadata.rb
name 'local_mode'
maintainer 'Dhruv Shah'
license 'All Rights Reserved'
description 'Run recipe in chef local mode'
version '0.1.0'
EOF

# Chef solo configuration
cat <<EOF > /root/solo.rb
cookbook_path ['/root/cookbooks']
EOF

# Execute the recipe with chef-client in local mode
chef-client -z -c /root/solo.rb -o 'local_mode::hello_file' --chef-license accept
```

### 📦 Step 2: add this `main.tf`
```hcl
resource "aws_instance" "local_mode_server" {
  ami                         = "ami-0f58b397bc5c1f2e8" # Ubuntu 20.04
  instance_type               = "t2.micro"
  key_name                    = aws_key_pair.deployer.key_name
  vpc_security_group_ids      = [aws_security_group.apache_sg.id]
  associate_public_ip_address = true

  user_data = file("install_local_mode.sh")

  tags = {
    Name = "Chef-LocalMode-Server"
  }
}
```

Apply with:
```bash
terraform apply -auto-approve
```

### 🧪 Verify on EC2
```bash
ssh -i ~/.ssh/id_rsa ubuntu@<EC2_PUBLIC_IP>
```

Check if the file was created by the Chef recipe:
```bash
sudo cat /root/hello_from_chef.txt
```

Output:
```bash
Hello, Chef is working in local mode!
```

---

## 🧩 Task 6: Create a Cookbook with Multiple Recipes and Attributes
### 📄 Step 1: Create `install_multi_recipe.sh`
This script installs Chef, creates a cookbook `multi_demo` with:
- 📁 `recipes/install_apache.rb`: installs and enables Apache
- 📁 `recipes/configure_index.rb`: writes content to `/var/www/html/index.html` using attributes
- 📁 `attributes/default.rb`: defines the homepage message

```bash
#!/bin/bash -xe

# Update & install packages
apt update -y
apt install -y curl unzip apache2-utils

# Install Chef Infra Client
curl -L https://omnitruck.chef.io/install.sh | bash

# Create cookbook structure
mkdir -p /root/cookbooks/multi_demo/{recipes,attributes}

# Define an attribute
cat <<EOF > /root/cookbooks/multi_demo/attributes/default.rb
default['multi_demo']['homepage_message'] = 'Welcome to Dhruv\'s Chef Demo!'
EOF

# Recipe 1: Install Apache
cat <<EOF > /root/cookbooks/multi_demo/recipes/install_apache.rb
package 'apache2' do
  action :install
end

service 'apache2' do
  action [:enable, :start]
end
EOF

# Recipe 2: Configure homepage using attribute
cat <<EOF > /root/cookbooks/multi_demo/recipes/configure_index.rb
file '/var/www/html/index.html' do
  content node['multi_demo']['homepage_message']
  owner 'root'
  group 'root'
  mode '0644'
  action :create
end
EOF

# Metadata
cat <<EOF > /root/cookbooks/multi_demo/metadata.rb
name 'multi_demo'
maintainer 'Dhruv Shah'
license 'All Rights Reserved'
description 'Chef cookbook with multiple recipes and attributes'
version '0.1.0'
EOF

# Chef Solo configuration
cat <<EOF > /root/solo.rb
cookbook_path ['/root/cookbooks']
EOF

# Execute both recipes
chef-client -z -c /root/solo.rb -o 'multi_demo::install_apache,multi_demo::configure_index' --chef-license accept
```

### 📦 Step 2: add this `main.tf`
```hcl
resource "aws_instance" "multi_recipe_server" {
  ami                         = "ami-0f58b397bc5c1f2e8"
  instance_type               = "t2.micro"
  key_name                    = aws_key_pair.deployer.key_name
  vpc_security_group_ids      = [aws_security_group.apache_sg.id]
  associate_public_ip_address = true

  user_data = file("install_multi_recipe.sh")

  tags = {
    Name = "Chef-Multi-Recipe-Server"
  }
}
```
Apply with:
```bash
terraform apply -auto-approve
```

## 🧪 Verify on EC2

SSH into your instance:
```bash
ssh -i ~/.ssh/id_rsa ubuntu@<EC2_PUBLIC_IP>
cat /var/www/html/index.html
```

**Expected Output:**
```
Welcome to Dhruv's Chef Demo!
```

