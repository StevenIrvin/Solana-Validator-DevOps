# Solana Validator DevOps

This project provides an Ansible-based automation suite for deploying and configuring a Firedancer validator node for the Solana blockchain. Firedancer is a high-performance validator client developed by Jump Crypto, and this repository streamlines its installation, configuration, and maintenance on Ubuntu-based servers.

## Purpose

The goal is to automate:
- Cloning and building Firedancer from source.
- Configuring system settings (e.g., `sysctl`, `hugetlbfs`, `ethtool`) for optimal performance.
- Setting up Git credentials and SSH keys.
- Managing environment-specific configurations (e.g., `test` vs. `main`).
- Rebooting the server and ensuring it comes back online.

This is designed for developers, DevOps engineers, or validators who want a repeatable, versioned setup for Solana validator nodes.

---

## Project Structure

```
Solana-Validator-DevOps/
├── firedancer-update.yaml        # Main playbook orchestrating the setup
├── inventory.yaml                # Inventory file with hosts and global variables
├── playbooks/
│   ├── firedancer/
│   │   ├── build.yaml           # Builds Firedancer and configures system
│   │   ├── git.yaml             # Sets up Git and SSH keys
│   │   ├── main.yaml            # (Placeholder; currently empty)
│   │   └── source.yaml          # Clones and updates Firedancer source
│   └── vars/
│       └── main.yaml            # Configures bash aliases and PS1 prompt
├── files/                       # (Recommended) Directory for SSH keys
└── README.md                    # This file
```

## Prerequisites

### System Requirements
- **OS**: Ubuntu (tested on 20.04/22.04).
- **User**: A user with sudo privileges (default: `ubuntu` for SSH, `solana` for Firedancer).
- **Hardware**: Sufficient resources for a Solana validator (e.g., 16+ CPU cores, 64GB+ RAM, NVMe SSD).

### Software Requirements
- **Ansible**: Installed on the control node (e.g., your local machine or a CI/CD server).
    - Install with: `pip install ansible` or `sudo apt install ansible`.
- **SSH Access**: Configured for the target hosts with a private key (default: `~/.ssh/id_rsa`).
- **Git**: Installed on the target hosts (handled by the playbook).

### SSH Key Setup
- An SSH key pair named `agave-deploy` (private key) and `agave-deploy.pub` (public key) is required for Git operations.
- Place these in `~/.ssh/` on the control node or in a `files/` directory in the project:

```
~/.ssh/
├── agave-deploy
└── agave-deploy.pub
```

### Validator Keys
- Validator keypairs (`validator-keypair.json`, `vote-account-keypair.json`) for each environment (`testnet`, `mainnet`) must be available on the control node at:

```bash
~/sln/validator/<root_name>/<env>/
```

## Setup Instructions

### 1. Clone the Repository
```bash
git clone <repository-url> Solana-Validator-DevOps
cd Solana-Validator-DevOps
```

### 2. Configure the Inventory
Edit inventory.yaml to match your environment:

```yaml
all:
  vars:
    git-email: "your-email@example.com"  # Your Git email
    git-name: "Your Name"                # Your Git username
  children:
    test:
      hosts:
        Test1:
          root_name: test1
          ansible_host: <test-ip>          # Replace with actual IP
          ansible_user: ubuntu
          env: test
          ansible_ssh_private_key_file: ~/.ssh/id_rsa
          version: "0.403.20113"           # Firedancer version
    main:
      hosts:
        Main1:
          root_name: main1
          ansible_host: <main-ip>          # Replace with actual IP
          ansible_user: ubuntu
          env: main
          ansible_ssh_private_key_file: ~/.ssh/id_rsa
          version: "0.403.20114"           # Firedancer version
```


- Replace <test-ip> and <main-ip> with your server IPs.
- Update git-email and git-name to your credentials.
- Adjust version to the desired Firedancer release tag (e.g., 0.403.20113)

### 3. Verify Inventory

Check that Ansible can parse the inventory:

```bash
ansible-inventory -i inventory.yaml --list
```

### 4. Test Connectivity

Ensure SSH works for all hosts:

```bash
ansible all -i inventory.yaml -m ping
```

If this fails, verify your SSH key and network access.


## Usage
Run the Playbook
Deploy Firedancer to all hosts:

```bash
ansible-playbook -i inventory.yaml firedancer-update.yaml
```

### What It Does
1) Source: Clones Firedancer from GitHub, checks out the specified version, and updates submodules.
1) Git: Configures SSH keys and Git credentials for the solana user.
1) Vars: Sets up bash aliases and a custom PS1 prompt based on the env variable.
1) Build: Installs dependencies, builds Firedancer, links binaries, configures system settings, and reboots.

### Options

Limit to a Host: Run on a specific host:

```bash
ansible-playbook -i inventory.yaml firedancer-update.yaml -l Test1
```

Verbose Output: Add `-v` for debugging:
```bash
ansible-playbook -i inventory.yaml firedancer-update.yaml -v
```

## Key Features

### Environment-Specific Config

- Creates `/etc/firedancer/config.toml` based on env
- Creates systmd service files for Firedancer based on env

### Transfers Validator Keys

- Deploys keys to /etc/firedancer/:
  - staked-identity.json to `/etc/firedancer/staked-identity.json`
  - vote-account-keypair.json tp `/etc/firedancer/vote-account-keypair.json`

### Environment-Specific Prompt
- The PS1 prompt in .bashrc changes color based on env:
  - Red for main (intended as mainnet).
  - Green for test.

### Bash Aliases

Added to `/home/solana/.bashrc`:

- catchup: solana catchup --our-localhost
- pubkey: fdctl keys pubkey
- monitor: sudo fdctl monitor

### Firedancer Configuration
Post-reboot, the playbook runs:

- fdctl configure init sysctl
- fdctl configure init hugetlbfs
- fdctl configure init ethtool-channels
- fdctl configure init ethtool-gro
- fdctl configure init ethtool-loopback
- fdctl configure init hyperthreads

## Post-Deployment

### Verify Installation
1) SSH into the host as ubuntu, then switch to solana:

```bash
ssh ubuntu@<host-ip>
sudo -i -u solana
```

1) Check Firedancer Version:

```bash
fdctl --version
```
