# Complete Guide to SELinux Configuration on CentOS 7

**🛡️ What is SELinux?** SELinux (Security-Enhanced Linux) is a kernel-level mandatory access control (MAC) system that provides an additional layer of security. It was developed by the US National Security Agency (NSA) and released as open source. SELinux enforces security policies that limit what processes and users can do, even if they have root privileges.

**⚠️ Important:** SELinux is a critical security component. Disabling it reduces system security. This guide uses CentOS 7 for all examples.

## Checking SELinux Status

### Using sestatus Command

/usr/sbin/sestatus

Example output:

SELinux status: enabled SELinuxfs mount: /sys/fs/selinux SELinux root directory: /etc/selinux Loaded policy name: targeted Current mode: enforcing Mode from config file: enforcing Policy MLS status: enabled Policy deny\_unknown status: allowed Memory protection checking: actual (secure) Max kernel policy version: 31

| Field | Description |
| --- | --- |
| `SELinux status: enabled` | Indicates that SELinux is enabled |
| `Current mode: enforcing` | SELinux is in enforced mode (actively blocking) |
| `Policy from config file: targeted` | Using targeted policy (protects specific services) |

### Using getenforce Command

/usr/sbin/getenforce

Possible outputs:

| Output | Meaning |
| --- | --- |
| `Enforcing` | SELinux is active and enforcing policy rules (blocking violations) |
| `Permissive` | SELinux is active but only logs violations (doesn't block) - only DAC rules are used |
| `Disabled` | SELinux is completely disabled |

## Installing and Verifying SELinux Packages

### Check Installed SELinux Packages

\# Check SELinux packages rpm -qa | grep selinux # Check policycoreutils rpm -q policycoreutils # Check setroubleshoot rpm -qa | grep setroubleshoot

Required packages:

*   `selinux-policy-targeted` - Targeted SELinux policy
*   `selinux-policy` - SELinux policy core
*   `libselinux` - SELinux library
*   `libselinux-python` - Python bindings for SELinux
*   `libselinux-utils` - SELinux utilities
*   `policycoreutils` - Policy core utilities
*   `setroubleshoot` - SELinux troubleshoot tools
*   `setroubleshoot-server` - Server for setroubleshoot
*   `setroubleshoot-plugins` - Plugins for setroubleshoot

#### Install Missing Packages:

sudo yum install selinux-policy-targeted selinux-policy libselinux libselinux-python \\ libselinux-utils policycoreutils setroubleshoot setroubleshoot-server setroubleshoot-plugins -y

## Initial SELinux Setup Process

**⚠️ Critical Step:** Before fully enabling SELinux, you must perform a proper initialization process to avoid system boot problems.

### Step 1: Set SELinux to Permissive Mode

Edit the SELinux configuration file:

sudo vi /etc/selinux/config

\# This file controls the state of SELinux on the system. # SELINUX= can take one of these three values: # enforcing - SELinux security policy is enforced. # permissive - SELinux prints warnings instead of enforcing. # disabled - No SELinux policy is loaded. SELINUX=permissive # SELINUXTYPE= can take one of these three values: # targeted - Targeted processes are protected, # minimum - Modification of targeted policy. Only selected processes are protected. # mls - Multi Level Security protection. SELINUXTYPE=targeted

**📌 Why permissive first?** Setting to permissive allows SELinux to label all files without blocking any services, preventing boot failures.

### Step 2: File System Relabeling

Before enabling SELinux, all files must be marked with SELinux contexts. Create the `.autorelabel` file:

sudo touch /.autorelabel

**⚠️ CRITICAL:** Without proper file labeling, limited domains may be denied access, which can lead to incorrect operating system loading or boot failures!

### Step 3: Reboot the System

sudo shutdown -r now # or sudo reboot

During this reboot:

*   The system will automatically relabel all files with SELinux contexts
*   This may take several minutes depending on the number of files
*   The system will boot with SELinux in permissive mode (logging only, not blocking)

🔄 System rebooting with file relabeling in progress...

### Step 4: Verify No SELinux Denials

After reboot, check if SELinux blocked any actions during boot:

sudo grep "SELinux is preventing" /var/log/messages

The output should be **empty**, which means SELinux did not block any actions and everything is working correctly.

✅ If empty: Everything is fine, proceed to enforcing mode.  
❌ If denials appear: Troubleshoot the denials before proceeding to enforcing.

### Step 5: Enable Enforcing Mode

After confirming no denials, set SELinux to enforcing mode:

sudo vi /etc/selinux/config

SELINUX=enforcing SELINUXTYPE=targeted

Or set it immediately without reboot:

sudo setenforce 1

Verify the change:

getenforce # Should output: Enforcing

### Step 6: Final Reboot

sudo shutdown -r now

After reboot, verify SELinux is working correctly:

/usr/sbin/getenforce # Should show: Enforcing /usr/sbin/sestatus | grep "Current mode" # Should show: Current mode: enforcing

## SELinux Policy Types

**📋 Policy Location:** All policies are located in the `/etc/selinux/` directory.

| Policy | Description | Use Case |
| --- | --- | --- |
| **targeted** | Protects basic system services (web server, DHCP, DNS, etc.) but does not restrict all other programs. This is the **default and most commonly used** policy. | Most servers, general purpose systems |
| **strict** | The most stringent policy. Manages not only network services but also user programs. Every process is confined. | High-security environments, military, financial |
| **mls** (Multi-Level Security) | Contains not only rules but also various security levels. Implements a multi-level security system based on SELinux. | Government, classified data, multi-level secure environments |

## Essential SELinux Commands

| Command | Purpose |
| --- | --- |
| `getenforce` | Show current SELinux mode (Enforcing/Permissive/Disabled) |
| `setenforce 0` | Set SELinux to permissive mode (immediate, until reboot) |
| `setenforce 1` | Set SELinux to enforcing mode (immediate, until reboot) |
| `sestatus` | Display detailed SELinux status |
| `ls -Z` | View SELinux context of files |
| `ps -Z` | View SELinux context of processes |
| `id -Z` | View SELinux context of current user |
| `chcon` | Change SELinux context of a file |
| `restorecon` | Restore default SELinux context |
| `audit2why` | Analyze SELinux denial messages |
| `audit2allow` | Generate policy rules from denial messages |
| `seinfo` | Query SELinux policy |
| `sesearch` | Search SELinux policy rules |

## Common SELinux Scenarios and Solutions

### Scenario 1: Web Server (Apache) Can't Access Files

**Problem:** Apache (httpd) can't read files in /var/www/html/

\# Check SELinux context ls -Z /var/www/html/ # Fix by restoring default context sudo restorecon -Rv /var/www/html/ # Or manually change context sudo chcon -R -t httpd\_sys\_content\_t /var/www/html/

### Scenario 2: Changing Default Port for a Service

**Problem:** Changing Apache to run on port 8080 (non-standard)

\# Check what ports are allowed for httpd sudo semanage port -l | grep http # Add port 8080 to httpd port context sudo semanage port -a -t http\_port\_t -p tcp 8080

### Scenario 3: Analyzing SELinux Denials

**Problem:** Something is being blocked, need to understand why

\# Check audit logs for denials sudo grep "SELinux is preventing" /var/log/audit/audit.log # Use audit2why to analyze sudo cat /var/log/audit/audit.log | audit2why # Generate policy to allow (if needed) sudo cat /var/log/audit/audit.log | audit2allow -M mypol sudo semodule -i mypol.pp

### Scenario 4: FTP Upload Directory Issues

**Problem:** vsftpd users can't upload files

\# Allow FTP to read/write in home directories sudo setsebool -P ftp\_home\_dir on # Check current booleans sudo getsebool -a | grep ftp

## Working with SELinux Booleans

Booleans allow you to enable/disable specific SELinux rules without writing policy.

| Command | Description |
| --- | --- |
| `getsebool -a` | List all SELinux booleans and their current state |
| `setsebool httpd_can_network_connect on` | Enable boolean (temporary, until reboot) |
| `setsebool -P httpd_can_network_connect on` | Enable boolean permanently (-P flag) |
| `setsebool httpd_can_network_connect off` | Disable boolean |

### Common Useful Booleans:

| Boolean | Purpose |
| --- | --- |
| `httpd_can_network_connect` | Allow Apache to make network connections (proxies, APIs) |
| `httpd_can_sendmail` | Allow Apache to send email |
| `httpd_enable_homedirs` | Allow Apache to access user home directories |
| `ftp_home_dir` | Allow FTP to read/write files in user home directories |
| `allow_ftpd_full_access` | Give FTP daemon full access to the filesystem |
| `virt_use_nfs` | Allow virtual machines to use NFS storage |
| `samba_enable_home_dirs` | Allow Samba to share user home directories |

## Troubleshooting SELinux

### 1\. Check Logs

\# Main SELinux denial log sudo tail -f /var/log/audit/audit.log # Messages log sudo tail -f /var/log/messages | grep SELinux # For older systems sudo tail -f /var/log/secure

### 2\. Install and Use setroubleshoot

\# Install setroubleshoot if not already installed sudo yum install setroubleshoot setroubleshoot-server -y # Start the service sudo systemctl start setroubleshootd sudo systemctl enable setroubleshootd # Check sealert messages sudo sealert -a /var/log/audit/audit.log

### 3\. Generate Custom Policy for Denials

\# Generate policy module from denials sudo grep "SELinux is preventing" /var/log/audit/audit.log | audit2allow -M localpolicy # Install the generated policy sudo semodule -i localpolicy.pp # Verify it's loaded sudo semodule -l | grep localpolicy

## Complete SELinux Setup Script

#!/bin/bash # Complete SELinux Setup Script for CentOS 7 # Run with: sudo bash setup-selinux.sh echo "=== SELinux Setup Script ===" # Check if running as root if \[\[ $EUID -ne 0 \]\]; then echo "This script must be run as root!" exit 1 fi # Step 1: Install required packages echo "Installing SELinux packages..." yum install -y selinux-policy-targeted selinux-policy libselinux \\ libselinux-python libselinux-utils policycoreutils setroubleshoot \\ setroubleshoot-server setroubleshoot-plugins # Step 2: Check current status echo "Current SELinux status:" getenforce # Step 3: Set to permissive in config echo "Setting SELinux to permissive in config..." sed -i 's/^SELINUX=.\*/SELINUX=permissive/' /etc/selinux/config # Step 4: Create autorelabel echo "Creating .autorelabel for file relabeling..." touch /.autorelabel # Step 5: Reboot echo "System will reboot in 10 seconds to relabel files..." echo "After reboot, check logs and then enable enforcing mode." sleep 10 reboot

## Quick Reference Card

| Task | Command |
| --- | --- |
| Check status | `sestatus` or `getenforce` |
| Temporarily disable | `setenforce 0` |
| Temporarily enable | `setenforce 1` |
| Permanently disable | Edit `/etc/selinux/config` → `SELINUX=disabled` |
| View file context | `ls -Z` |
| View process context | `ps -Z` |
| Restore file context | `restorecon -Rv /path` |
| View booleans | `getsebool -a` |
| Set boolean permanently | `setsebool -P boolean on` |
| View audit logs | `tail -f /var/log/audit/audit.log` |
| Analyze denials | `audit2why < /var/log/audit/audit.log` |

**⚠️ Final Warning:**

*   Never disable SELinux permanently without understanding the security implications
*   Always test in permissive mode first before enforcing
*   Monitor logs regularly for denials
*   Use booleans and custom policies instead of disabling SELinux
*   Remember: SELinux is a critical security layer - work with it, not against it

**📚 Additional Resources:**

*   [Red Hat SELinux Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/)
*   `man selinux` - SELinux manual page
*   `man semanage` - SELinux management tool manual
*   `/etc/selinux/config` - Main configuration file
