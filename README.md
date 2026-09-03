# Ansible Multi-Tier Infrastructure Deployment

A production-style Ansible project that provisions and configures a load-balanced, multi-server web infrastructure on AWS — built to reflect how real infrastructure teams structure and automate deployments, not just a single-server tutorial.

## Overview

This project uses Ansible to configure a 3-server architecture on AWS EC2:

- **1 Load Balancer** — Nginx, reverse-proxying and round-robin distributing traffic across two backend servers, protected by HTTP Basic Auth
- **2 Application Servers** — Nginx serving app content, each identifiable by hostname to prove traffic is actually being distributed

All servers are discovered dynamically from AWS (no hardcoded IPs anywhere in the codebase), configured via reusable Ansible roles, and validated by a CI pipeline on every push.

## Architecture

                        ┌─────────────────┐
                        │   Internet       │
                        └────────┬─────────┘
                                 │  HTTP (Basic Auth)
                        ┌────────▼─────────┐
                        │   Load Balancer   │
                        │   (Nginx)         │
                        └────────┬─────────┘
                    round-robin  │
                 ┌───────────────┴───────────────┐
                 │                                │
        ┌────────▼─────────┐            ┌────────▼─────────┐
        │  App Server 1     │            │  App Server 2     │
        │  (Nginx)          │            │  (Nginx)          │
        └───────────────────┘            └───────────────────┘

    Control node (WSL2 + Ansible) configures all 3 over SSH


- **Control node**: WSL2 (Ubuntu), running Ansible — no agents installed on any target
- **Managed nodes**: 3x AWS EC2 instances (Ubuntu 24.04), discovered live via AWS's API
- **Connection**: SSH key-based auth

## Key features

- **Dynamic inventory** (`aws_ec2.yml`) — queries AWS directly for running instances and groups them by their `Name` tag, instead of a static list of IPs. Servers can be replaced or scaled without touching any config file.
- **Role-based structure** — configuration is split into `common`, `webserver`, and `loadbalancer` roles, each self-contained and independently reusable, rather than one flat script.
- **Load balancing** — the load balancer's Nginx config is generated from a Jinja2 template that pulls the app servers' live IPs straight from the dynamic inventory at run time.
- **Ansible Vault** — the Basic Auth password is encrypted at rest (`group_vars/lb_server/vault.yml`) and only decrypted in memory at deploy time, so the repo can be public without leaking secrets.
- **Idempotency** — every task is safe to re-run; a second run against unchanged infrastructure reports zero changes.
- **CI pipeline** (`.github/workflows/lint.yml`) — GitHub Actions runs `ansible-lint` at the `production` profile on every push and pull request, catching style and security issues before they reach `main`.


## Project structure
.
- ├── aws_ec2.yml                        # Dynamic inventory config (AWS EC2 plugin)
- ├── group_vars/
- │   ├── all.yml                        # SSH user/key, applied to all hosts
- │   └── lb_server/
- │       └── vault.yml                  # Vault-encrypted Basic Auth password
- ├── roles/
- │   ├── common/                        # Baseline setup applied to every server
- │   ├── webserver/                     # Installs Nginx, deploys app content
- │   │   └── templates/index.html.j2
- │   └── loadbalancer/                  # Configures Nginx as a reverse proxy
- │       └── templates/lb.conf.j2
- ├── site.yml                           # Top-level playbook tying roles to host groups
- └── .github/workflows/lint.yml         # CI: ansible-lint on every push

## Prerequisites

- Ansible installed on the control node, plus the `amazon.aws` and `community.general` collections
- Python `boto3`/`botocore` (required by the AWS dynamic inventory plugin)
- An AWS IAM user with `AmazonEC2ReadOnlyAccess` permissions, configured via `aws configure`
- 3 running EC2 instances tagged `Name: lb-server`, `Name: app-server-1`, `Name: app-server-2`, with SSH (22) and HTTP (80) open in their security groups
- An SSH key pair for authentication (not included in this repo)

## Usage

1. Configure AWS credentials: `aws configure`
2. Set your SSH key path in `group_vars/all.yml`
3. Confirm Ansible can see your infrastructure:
   ```bash
   ansible-inventory -i aws_ec2.yml --graph
   ```
4. Run the playbook (you'll be prompted for the Vault password):
   ```bash
   ansible-playbook -i aws_ec2.yml site.yml --ask-vault-pass
   ```
5. Visit `http://<load-balancer-public-ip>` in a browser — log in with the configured Basic Auth credentials, then refresh a few times to see traffic alternate between both app servers.

## What I learned building this

- The practical difference between a control node and managed nodes, and why Ansible's agentless (SSH-based) model matters
- Debugging real SSH key issues — permissions, malformed `.pem` files, missing trailing newlines — the kind of problems that don't show up in tutorials
- Writing reusable, idempotent roles instead of flat playbooks
- Building AWS dynamic inventory so infrastructure can change without editing code
- Using Jinja2 templates to generate configs (including an Nginx `upstream` block) from live inventory data
- Encrypting secrets at rest with Ansible Vault and wiring them into a real security control (HTTP Basic Auth)
- Setting up a CI pipeline so code quality is enforced automatically, not manually
- Managing AWS IAM users with least-privilege permissions, and handling credential rotation after an exposure
