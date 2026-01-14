# Hengamer03 Motd

![MOTD1 Screenshot](docs/Motd-1)

A dynamic MOTD system that displays fresh system information every time you open a new shell session.

## Key Features

- **🎨 Color Coding** - Visual status indicators (green = normal, yellow = warning, red = critical)
- **🔄 Reboot Detection** - Automatically detects when system restart is required after updates
- **📦 Container Monitoring** - Displays Docker/Podman container counts (optional, disabled by default)
- **❌ Failed Services** - Shows systemd services that have failed
- **✅ Custom Health Checks** - Define application-specific checks that display failures
- **📊 Real-time Metrics** - Collects system info at login time, not deployment time
- **📝 Ansible Tracking** - Shows when Ansible last ran on the server

## Requirements
- **OS**: RHEL or any RPM-based distribution (Fedora, Rocky Linux, AlmaLinux)
- **Ansible Version**: Tested on Ansible Core 2.14.18. Other versions may work but are not guaranteed.
- **Required Packages**: The role installs all required packages on the remote host.

## Quickstart

The role primarily works on variables are defined in `motd/defaults/main.yml` This makes it customizable to fit most ansible environments. The examples below will show how a `inventory/group_vars/all.yml` could look, and defines a basic setup.

```yml

## Server Identification

motd_env: "Production"
motd_datacenter: "Oslo-DC1"
motd_status: "Active"
motd_responsible: "ops@example.com"
motd_ops_team: "Infrastructure Team"
motd_backup: "Yes - Daily 02:00 UTC"

## Display Options

motd_banner_top: "Welcome to {{ motd_env }} ({{ motd_datacenter }})"
motd_show_cowsay: true

## Custom Health Checks

motd_health_checks:
  - name: "Web Server"
    check: "systemctl is-active httpd"
  
  - name: "Database"
    check: "systemctl is-active postgresql-14"
  
  - name: "Redis Cache"
    check: "systemctl is-active redis"
  
  - name: "Application Port"
    check: "ss -tuln | grep -q ':8080'"

## Adjustable Thresholds

motd_memory_warn_threshold: 75
motd_memory_crit_threshold: 90
motd_disk_warn_threshold: 75
motd_disk_crit_threshold: 90

## Optional packages

motd_install_reboot_tools: true
motd_install_container_tools: false
``` 

Run this role on any playbook by including the role

```yml
- hosts: all
  roles:
    - role: motd # make sure that ansible.cfg knows where to find its roles
```

## How It Works

This role deploys two components:

1. **Dynamic Script** (`/usr/local/bin/motd-status`) - Collects real-time system information
2. **Profile Hook** (`/etc/profile.d/00-motd.sh`) - Triggers the script on SSH login

Unlike static MOTD files, this setup runs the status script **every time you log in**, ensuring you always see current system state.

**Information Displayed:**
- Static metadata (from Ansible variables): Environment, datacenter, responsible team, backup config
- Dynamic metrics (collected at runtime): Load, memory, disk usage, uptime, running containers, failed services

## File Structure

```yml
.
├── defaults
│   └── main.yml
├── docs
│   ├── Motd-1 
│   └── motd-2
├── README.md
├── tasks
│   └── main.yml
├── templates
│   ├── motd_status.sh.j2
│   └── profile_motd.sh.j2
└── vars
    └── main.yml
```


## Configuration Variables

All variables are optional. The file `motd/defaults/main.yml` provides sensible defaults that you can override as needed.

### Server Identification
- ```motd_env``` : Display the environment the server is in. Dev, Test, QA, Production or whatever value you like
- ```motd_datacenter```: Location of the server. 
- ```motd_status```: Display the current server status. not dynamic but useful if you want to show if the server is under maintenance
- ```motd_responsible```: Responsible Team, what team owns the server
- ```motd_ops_team```: Operations Team 
- ```motd_backup```: Display the Backup Schema for the server

### Display Options
- ```motd_banner_top```: Display the message cowsay will display at the top
- ```motd_show_cowsay```: true / false , should cowsay be displayed or not

### Custom Health Checks

Define application-specific health checks that display when they fail. Each check runs a command that should exit with code 0 for success, non-zero for failure.

**Examples:**
```yaml
motd_health_checks:
  # Systemd service checks
  - name: "Web Server"
    check: "systemctl is-active httpd"
  
  - name: "Database"
    check: "systemctl is-active postgresql"
  
  # Network checks
  - name: "Application Port"
    check: "ss -tuln | grep -q ':8080'"
  
  # Custom script
  - name: "API Health"
    check: "/usr/local/bin/check-app-health.sh"
  
  # File existence
  - name: "Config File"
    check: "test -f /etc/myapp/config.yaml"
  
  # Disk space
  - name: "Disk Space /var < 90%"
    check: "test $(df /var | awk 'NR==2 {print $5}' | tr -d '%') -lt 90"
```

**Note:** Health checks execute every time you log in, so keep them fast (< 1 second).

### Adjustable Thresholds

These thresholds let you choose when warnings appear. You can be more lenient in development environments and more strict in production.


```yml
## Memory Checks
motd_memory_warn_threshold: 75
motd_memory_crit_threshold: 90

## Disk Checks
motd_disk_warn_threshold: 75
motd_disk_crit_threshold: 90
```

### Optional packages

The role can display if a server needs rebooting, or display containers. these are optional and can be turned off

```yml
motd_install_reboot_tools: true
motd_install_container_tools: false
``` 

## Example Output

Based on the screenshot, here's what you'll see:
```
   __________________________
  <  Welcome to Production  >
   --------------------------
         \   ^__^
          \  (oo)\_______
             (__)\       )\/\
                 ||----w |
                 ||     ||

  Name:        vm2.example.com         Subnet:      192.168.121.67/24
  IP:          192.168.121.67          Status:      Active
  Datacenter:  Oslo-DC1                Responsible: Infrastructure Team
  Environment: Production              Ops Team:    Infrastructure Team
  Backup:      Yes - Daily 02:00 UTC

  System status 2026-01-14 13:33:09+0000
  Load:   0.25 0.07 0.02     Boot:   Jan 14
  Procs:  124                Uptime: 7 hours, 12 minutes
  Memory: 0.4G/1.9G (22%)    Users:  2
  Swap:   0.0G/2.0G (0%)     Errors: 4
  Disk:
          /         2.7G   /70G    (4%)

  Ansible: Last run 0h ago
```

**With Warnings:**
```
  Memory: 7.2G/8.0G (90%)    ← Yellow warning
  Disk:
          /         94G   /100G   (94%)  ← Red critical

  ⚠ FAILED SERVICES: 2
  Services: httpd.service php-fpm.service

  ⚠ SYSTEM REBOOT REQUIRED
  Reason: New kernel installed

  ⚠ HEALTH CHECK FAILURES:
    ✗ Web Server
    ✗ API Health
```

**Photo example**

![MOTD2 Screenshot](docs/motd-2)

## Troubleshooting

### MOTD Not Displaying

1. Check if the script exists:
```bash
   ls -la /usr/local/bin/motd-status
```

2. Test the script manually:
```bash
   /usr/local/bin/motd-status
```

3. Verify profile.d hook:
```bash
   ls -la /etc/profile.d/00-motd.sh
```

### Health Checks Always Failing

Test the check command manually:
```bash
systemctl is-active httpd
echo $?  # Should be 0 for success
```

### Reboot Detection Not Working

Install reboot detection tools:
```bash
# RHEL 8+
dnf install -y dnf-utils

# RHEL 7
yum install -y yum-utils
```

Or set in your configuration:
```yaml
motd_install_reboot_tools: true
```

## License 

MIT

## Author

Created By: hengamer03 
Github: https://github.com/hengamer03

