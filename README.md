ample# Ansible Playbook for Self-Hosted LAN Infrastructure

This project automates the deployment of a robust, offline-first network infrastructure using Ansible. It sets up a central server to act as a DNS resolver, a reverse proxy, and a local Certificate Authority (CA), providing secure HTTPS access to all internal services.

This playbook is designed to be generic, configurable, and idempotent.

## Features

- **Centralized DNS**: Deploys **BIND9** as an authoritative DNS server for a local domain, allowing you to access services by name (e.g., `grafana.example.lan`).
- **Secure Reverse Proxy**: Deploys **Nginx** to act as a reverse proxy, providing a single entry point for all web services.
- **Automatic HTTPS**: Uses **mkcert** to create a private Certificate Authority (CA) and generate wildcard SSL certificates, enabling HTTPS for all internal services without browser warnings.
- **Offline First**: All required software packages are bundled, allowing the entire infrastructure to be deployed on servers with no internet access.
- **Highly Configurable**: All settings, including domain names, server IPs, and service definitions, are managed through simple variable files.
- **Idempotent & Testable**: The playbook is designed to be run multiple times without causing issues and can be tested locally without requiring remote servers.

## Project Structure

```
.
├── ansible.cfg         # Ansible configuration
├── inventory           # List of your servers
├── playbook.yml        # The main playbook orchestrating all roles
├── README.md           # This file
├── offline_files/      # Directory for offline software packages
└── roles/              # Directory containing all Ansible roles
    ├── bind/           # Role for BIND9 setup
    ├── mkcert/         # Role for local CA and certificate generation
    ├── nginx/          # Role for Nginx reverse proxy setup
    └── dns_client/     # Role to configure client machines to use the new DNS
```

## Prerequisites

1.  **Ansible Control Node**: A machine (e.g., your laptop) with `ansible-core` installed. This is the machine you will run the playbook from.
2.  **Target Servers**: One or more clean servers (Ubuntu/Debian-based) with SSH access configured.
3.  **Offline Packages**: You must download the necessary software packages on an internet-connected machine and place them in the `offline_files/` directory.

---

## How to Use: A Step-by-Step Guide

### Step 1: Prepare Offline Packages

On a machine with internet access (and the same OS/version as your target servers), run the following commands to download the required packages:

```bash
# Create a temporary directory
mkdir -p temp_packages && cd temp_packages

# Download BIND9, Nginx, and their dependencies
sudo apt-get update
sudo apt-get --download-only -o Dir::Cache::archives="./" install bind9 bind9utils dnsutils nginx

# Package them into tarballs
tar -czvf bind9-offline-packages.tar.gz bind9*.deb dnsutils*.deb ... # (include all downloaded .deb files)
tar -czvf nginx-offline-package.tar.gz nginx*.deb ...

# Download mkcert
wget https://github.com/FiloSottile/mkcert/releases/download/v1.4.4/mkcert-v1.4.4-linux-amd64 -O mkcert
chmod +x mkcert
tar -czvf mkcert.tar.gz mkcert

# Move the final tarballs to the project's offline_files directory
mv *.tar.gz /path/to/your/ansible-sirat-infra/offline_files/
```

### Step 2: Configure Your Infrastructure

You only need to edit **two files** to customize the entire deployment.

#### A. Configure `inventory`

Open the `inventory` file and replace the placeholder IPs with the **actual IP addresses** of your servers.

```ini
# Example:
[dns_proxy_server]
dns-proxy ansible_host=192.168.1.10

[application_servers]
app-server-01 ansible_host=192.168.1.20
app-server-02 ansible_host=192.168.1.21
```

#### B. Configure `playbook.yml`

Open the `playbook.yml` file and edit the `vars` section.

1.  **`domain_name`**: Change `"example.lan"` to your desired internal domain name (e.g., `"sirat.lan"`).
2.  **`app_server_ips`**: Map the hostnames from your inventory to their IP addresses.
3.  **`services`**: This is the most important part. Define all your internal services here. Add, remove, or modify entries as needed. The playbook will automatically generate DNS records and Nginx configurations based on this list.

```yaml
# Example vars from playbook.yml
vars:
  domain_name: "example.lan"

  app_server_ips:
    app-server-01: "192.168.1.20"
    app-server-02: "192.168.1.21"

  services:
    - { name: "grafana", target_host: "app-server-01", port: 3000 }
    - { name: "vault", target_host: "app-server-02", port: 8200 }
```

### Step 3: Run the Playbook

Once your configuration is ready, run the playbook from the root directory of the project:

```bash
ansible-playbook playbook.yml
```

Ansible will connect to your servers, install the packages, and configure everything automatically.

---

## Step 4: Trust the Certificate Authority (CA) on Client Machines

This is a **critical final step**. After the playbook runs, a new private Certificate Authority is created. Your computer's browser does not know about this CA, so you must manually tell it to trust it. If you skip this step, you will see security warnings in your browser.

You must perform these actions on **every machine** (your laptop, your colleagues' machines, etc.) that needs to access the HTTPS services.

### A. Get the Root CA Certificate

First, you need to retrieve the public certificate of your new CA.

1.  SSH into your DNS/Proxy server (the one with IP `192.168.1.10` in our example).
2.  Run the following command to display the certificate content:
    ```bash
    sudo cat $(mkcert -CAROOT)/rootCA.pem
    ```
3.  Copy the entire output, including the `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----` lines.

### B. Install the Root CA Certificate

Now, save the copied content into a file named `my-lan-ca.pem` on your local machine and follow the instructions for your operating system.

#### On Linux (Ubuntu, Debian, etc.)

1.  Move the certificate to the system's CA directory.
    ```bash
    # (Assuming my-lan-ca.pem is in your current directory)
    sudo cp my-lan-ca.pem /usr/local/share/ca-certificates/my-lan-ca.crt
    ```
2.  Update the system's certificate store.
    ```bash
    sudo update-ca-certificates
    ```
    You should see a message confirming that 1 certificate was added.

#### On Windows

1.  Double-click the `my-lan-ca.pem` file.
2.  Click the **"Install Certificate..."** button.
3.  Choose **"Local Machine"** and click Next.
4.  Select **"Place all certificates in the following store"**.
5.  Click **"Browse..."** and select the **"Trusted Root Certification Authorities"** store. Click OK.
6.  Click Next, then Finish. Acknowledge the security warning by clicking **Yes**.

#### On macOS

1.  Double-click the `my-lan-ca.pem` file to open it in **Keychain Access**.
2.  Find the certificate in the "login" keychain. It will likely have a red "X" icon, indicating it is not trusted.
3.  Double-click the certificate to open its details.
4.  Expand the **"Trust"** section.
5.  Change the "When using this certificate" dropdown to **"Always Trust"**.
6.  Close the window. You will be prompted for your password to save the changes.

After completing these steps, **restart your web browser**. You should now be able to access all your services (e.g., `https://grafana.sirat.lan`) with a valid HTTPS connection and a green lock icon.
