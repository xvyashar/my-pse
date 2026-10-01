# My Personal Server Environment
My personal vps server setup ansible playbook

## Usage
Rename `secrets.ini.example` file to `secrets.ini` and fill it with your values.

### First run:
```bash
$ ansible-playbook site.yml -i inventory.ini -i secrets.ini --ask-pass
```

### After once bootstrap tag executed:
```bash
$ ansible-playbook site.yml -i inventory.ini -i secrets.ini --skip-tags bootstrap
```

## Plays
- **Bootstrap**: init vps server by creating a sudo user, updating and upgrading packages, installing ufw firewall and fail2ban, enabling ssh key login on control node (usually your desktop), and finally disabling root login