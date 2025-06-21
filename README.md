# 🧪 Task 1: Install Chef Workstation on Local Machine (Windows)

## 📌 Objective
Install [Chef Workstation](https://community.chef.io/downloads/tools/workstation?os=ubuntu) on your **local Windows machine** to use tools like `chef`, `knife`, `chef-solo`, etc., from your terminal.

---

## ✅ Step-by-Step Instructions
### 🔧 Step 1: Download the Official Chef Workstation Installer

1. Visit the official download page:  
   👉 https://community.chef.io/downloads/tools/workstation?os=ubuntu

2. Select:
   - **Platform:** Windows
   - **Architecture:** x86_64

3. Click **Download** to get the `.msi` installer file.

---

### 🔧 Step 2: Install Chef Workstation

1. **Double-click** the downloaded `.msi` file.
2. Follow the installer steps:
   - Accept the license
   - Use the default installation directory
   - Click **Install**
3. Once complete, click **Finish**

---

### 🔄 Step 3: Restart Terminal

- **Close and reopen** PowerShell or Git Bash.

---

### 🔍 Step 4: Verify the Installation

Open **PowerShell** and run:

```powershell
chef -v
```
You should see output like:
```
Chef Workstation version: 23.x.x
Chef Infra Client version: 18.x.x
```

# 🛠️ Task 2: Launch EC2 & Install Apache via Chef Using Terraform
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
```bash
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
```
terraform init
terraform apply -auto-approve
```

🔍 Step 4: Verify Apache Web Server
Go to AWS EC2 console → find your instance.

Copy the Public IP.

Visit in browser:
```
http://<EC2_PUBLIC_IP>
```
✅ You should see the Apache2 Ubuntu Default Page.
