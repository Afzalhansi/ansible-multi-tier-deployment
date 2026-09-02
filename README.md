# Ansible Nginx EC2 Deployment

My first Ansible project — using Ansible to configure a remote AWS EC2 instance as a basic web server running Nginx.

## What this does

Running a single playbook against a fresh EC2 instance will:
- Update the apt package cache
- Install Nginx
- Deploy a custom `index.html` page
- Ensure Nginx is running and enabled on boot

## Architecture

- **Control node**: WSL2 (Ubuntu) running Ansible
- **Managed node**: AWS EC2 instance (Ubuntu 24.04), configured entirely over SSH — no agent installed on the target
- **Connection**: SSH key-based auth (`.pem` key, excluded from this repo via `.gitignore`)

## Project structure

.
├── inventory.ini # Defines the target host(s)
├── playbook.yml # Defines the desired configuration
└── .gitignore # Excludes private keys and Ansible runtime files


## Prerequisites

- Ansible installed on the control node
- An EC2 instance with SSH access (port 22) and HTTP access (port 80) open in its security group
- A `.pem` key file for SSH authentication (not included — provide your own)

## Usage

1. Update `inventory.ini` with your EC2 instance's public IP and key path.
2. Run:
```bash
   ansible-playbook -i inventory.ini playbook.yml
```
3. Visit `http://<your-ec2-public-ip>` in a browser to confirm the deployment.

## What I learned

- Ansible's control node vs. managed node architecture
- SSH key-based authentication and troubleshooting (permissions, key formatting)
- Writing a basic idempotent playbook using the `apt`, `copy`, and `service` modules
- Managing AWS EC2 security group rules for SSH and HTTP access
