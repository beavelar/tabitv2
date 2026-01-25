# TabitV2 Minecraft Server Setup

Ansible playbook to consistently setup the TabitV2 Minecraft server. Manual setup steps can be viewed through the [`setup`](setup.md) steps. Steps below are to setup with the aid of Ansible

## Prerequisites

- [`Ansible`](https://docs.ansible.com/projects/ansible/latest/installation_guide/intro_installation.html)

## Setup

1. In the host machine, generate a SSH key for each account (`beamc`, `beamcs`, and `beaprom`)

   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_hetz_beamc
   ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_hetz_beamcs
   ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_hetz_beaprom
   ```

2. Create or edit the `~/.ssh/config` file, paste the following content

   ```
   Host hetz-root
       HostName <ip-address>
       User root

    Host hetz-beamc
       HostName <ip-address>
       User beamc
       IdentityFile ~/.ssh/id_ed25519_hetz_beamc
       Port 1738

    Host hetz-beamcs
       HostName <ip-address>
       User beamcs
       IdentityFile ~/.ssh/id_ed25519_hetz_beamcs

    Host hetz-beaprom
       HostName <ip-address>
       User beaprom
       IdentityFile ~/.ssh/id_ed25519_hetz_beaprom
   ```

   _Ensure to change `<ip-address>` to the IP address of the remote machine_

3. Create a Ansible vault

   ```bash
   ansible-vault create group_vars/vault.yml
   ```

4. In the vault, set the following values

   ```
   beamcs_password: "<beamcs-password>"
   ```

   _Ensure to replace `<beamcs-password>` with the account password_

5. Run the Ansible base_setup playbook run command

   ```bash
   ansible-playbook -i inventory.ini playbooks/base_setup.yml --ask-pass --ask-vault-pass
   ```

   _If you haven't already SSH'd into the remote machine, do so at least once to add the host's fingerprint to your known_hosts file_

6. Setup Cloudflare

   a. In Cloudflare, select the domain for the Minecraft server

   b. Navigate to DNS -> Records

   c. Add a new record with the following

   ```
   - Type: A

   - Name (required): minecraft.tabitv2.com

   - IPv4 address (required): Server IP address

   - Proxy status: Disabled
   ```

   d. Save

   e. Add a new record with the following

   ```
   - Type: A

   - Name (required): grafana.tabitv2.com

   - IPv4 address (required): Server IP address

   - Proxy status: Enabled
   ```

   f. Save

7. Run the Ansible setup playbook run command

   ```bash
   ansible-playbook -i inventory.ini playbooks/setup.yml --ask-become-pass --ask-vault-pass
   ```
