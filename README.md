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
...

```
