# LocalWP Setup Guide for Crostini (Chrome OS Linux)

This guide provides step-by-step instructions for installing and configuring LocalWP to run successfully in a Crostini Linux environment on Chrome OS.

## Overview

LocalWP is a local WordPress development environment that typically faces several challenges when running in Crostini due to privilege escalation requirements and containerized environment limitations. This guide addresses all known issues and provides working solutions.

## Prerequisites

- Chrome OS with Linux (Crostini) enabled
- Terminal access to the Linux environment
- Basic familiarity with command line operations

## Issues Addressed

1. **Missing dependencies** (pkexec, polkit authentication)
2. **Polkit authentication agent failures** in GUI applications
3. **MySQL configuration and initialization problems**
4. **Nginx privilege binding issues**
5. **Service configuration template processing**

## Step-by-Step Installation

### 1. System Preparation

First, update your system and install required dependencies:

```bash
# Update package repositories
sudo apt update

# Install required dependencies
sudo apt install -y wget dpkg libaio1 libncurses5 libnss3-tools policykit-1 policykit-1-gnome libcap2-bin
```

### 2. Configure Polkit Authentication (CRITICAL - Do This FIRST!)

**⚠️ IMPORTANT:** Configure polkit authentication BEFORE installing LocalWP to prevent site creation failures and configuration issues.

#### 2.1 Create Polkit Rules

Create a custom polkit rule that allows LocalWP operations without authentication prompts:

```bash
sudo tee /etc/polkit-1/rules.d/99-localwp-allow.rules > /dev/null << 'EOF'
/* Allow LocalWP operations without authentication */
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.policykit.exec" &&
        subject.user == "YOUR_USERNAME") {
        return polkit.Result.YES;
    }
});

polkit.addRule(function(action, subject) {
    if (action.id.indexOf("org.freedesktop") == 0 &&
        subject.user == "YOUR_USERNAME") {
        return polkit.Result.YES;
    }
});

polkit.addRule(function(action, subject) {
    if (subject.user == "YOUR_USERNAME" && subject.local) {
        return polkit.Result.YES;
    }
});
EOF
```

**Important:** Replace `YOUR_USERNAME` with your actual username. You can find it with:
```bash
echo $USER
```

#### 2.2 Reload Polkit Rules

```bash
# Reload polkit daemon to apply new rules
sudo pkill -HUP polkitd

# Verify the rules are loaded (should show 8 rules if successful)
sudo journalctl -u polkit -n 5
```

#### 2.3 Test Polkit Configuration

```bash
# This should work without prompting for a password
pkexec echo "Testing pkexec with polkit rules"
```

**⚠️ CRITICAL:** If this doesn't work without a password prompt, do NOT proceed. Double-check your username in the polkit rules and fix this first.

### 3. Download LocalWP

Now that polkit is properly configured, get the latest LocalWP Linux package:

```bash
# Download LocalWP (check for latest version at https://localwp.com/releases/)
wget -O local-linux.deb "https://cdn.localwp.com/stable/latest/deb"

# If the above fails, use the direct version link:
# wget -O local-linux.deb "https://cdn.localwp.com/releases-stable/9.2.4+6788/local-9.2.4-linux.deb"
```

### 4. Install LocalWP

Install the downloaded package and fix any dependency issues:

```bash
# Install LocalWP
sudo dpkg -i local-linux.deb

# Fix any broken dependencies
sudo apt --fix-broken install -y
```

### 5. Configure Nginx Capabilities

Set the required network binding capabilities for Nginx:

```bash
# Set capability for Nginx to bind to privileged ports
sudo setcap CAP_NET_BIND_SERVICE=+eip ~/.config/Local/lightning-services/nginx-*/bin/linux/sbin/nginx

# Verify the capability was set
/usr/sbin/getcap ~/.config/Local/lightning-services/nginx-*/bin/linux/sbin/nginx
```

**Note:** This step may need to be repeated after LocalWP updates that replace the Nginx binary.

### 6. Launch LocalWP and Create Sites

With polkit properly configured, LocalWP should now work correctly:

```bash
# Launch LocalWP with GPU and sandbox disabled (recommended for Crostini)
/opt/Local/local --no-sandbox --disable-gpu
```

**Expected Result:** Site creation should now complete successfully without authentication prompts or configuration errors.

### 7. Backup Plan: Manual MySQL Configuration (If Needed)

**Note:** With polkit configured first, you likely won't need this section. However, if MySQL still fails to start, manually create the configuration:

#### 7.1 Locate Your Site Directory

```bash
# Find your site's run directory (replace SITE_ID with actual directory name)
ls ~/.config/Local/run/
# Example output: yMIPhb2Uu (this is your SITE_ID)
```

#### 7.2 Create MySQL Configuration

```bash
# Replace SITE_ID with your actual site directory name
SITE_ID="your_site_id_here"

# Create MySQL configuration
mkdir -p ~/.config/Local/run/$SITE_ID/conf/mysql
cat > ~/.config/Local/run/$SITE_ID/conf/mysql/my.cnf << EOF
[mysqld]
skip-name-resolve

datadir = /home/$USER/.config/Local/run/$SITE_ID/mysql/data
port = 10004
bind-address = 127.0.0.1
socket = /home/$USER/.config/Local/run/$SITE_ID/mysql/mysqld.sock

# Older PHP/client compatibility
character-set-server = utf8mb3
default_authentication_plugin = mysql_native_password

# Fine Tuning
performance_schema = off
max_allowed_packet = 16M
thread_stack = 192K
thread_cache_size = 8

# InnoDB
innodb_buffer_pool_size = 32M
innodb_log_file_size = 96M

[client]
socket = /home/$USER/.config/Local/run/$SITE_ID/mysql/mysqld.sock
user = root
password = root
EOF
```

#### 7.3 Initialize MySQL Database

```bash
# Create data directory
mkdir -p ~/.config/Local/run/$SITE_ID/mysql/data

# Initialize MySQL database
~/.config/Local/lightning-services/mysql-*/bin/linux/bin/mysqld \
  --defaults-file=~/.config/Local/run/$SITE_ID/conf/mysql/my.cnf \
  --initialize-insecure
```

### 8. Optional: Fix PHP-FPM Configuration (If Needed)

**Note:** With polkit configured first, you likely won't need this section. However, if PHP-FPM fails to start, create its configuration:

```bash
# Create PHP-FPM configuration
mkdir -p ~/.config/Local/run/$SITE_ID/conf/php
cat > ~/.config/Local/run/$SITE_ID/conf/php/php-fpm.conf << EOF
[global]
error_log = "/home/$USER/.config/Local/run/$SITE_ID/logs/php-fpm.log"
daemonize = no
include=php-fpm.d/*.conf
EOF
```

## Why Order Matters

**Important Insight:** Configuring polkit authentication BEFORE installing LocalWP prevents most configuration issues because:

1. **Complete Site Creation:** LocalWP can complete the entire site creation process without interruption
2. **Template Processing:** Configuration templates get processed properly into actual config files
3. **Service Initialization:** All services (MySQL, PHP-FPM, Nginx) get configured automatically
4. **No Manual Fixes:** You avoid having to manually create configuration files

**Previous Problem:** When polkit wasn't configured first, LocalWP's site creation process would fail partway through due to authentication errors, leaving sites in a partially-created state with missing configuration files.

## Troubleshooting

### Common Issues and Solutions

#### 1. "No polkit authentication agent found" Error

**Solution:** Ensure you've completed Step 4 (Configure Polkit Authentication) and that your username is correct in the polkit rules.

#### 2. MySQL Connection Failures

**Solution:** Follow Step 7 to manually create and initialize the MySQL configuration.

#### 3. Nginx Permission Denied

**Solution:** Re-run the setcap command from Step 5 after any LocalWP updates.

#### 4. GUI Not Appearing

**Solution:** Always use the `--no-sandbox --disable-gpu` flags when launching LocalWP in Crostini.

### Verification Commands

Test individual components:

```bash
# Test polkit
pkexec echo "Polkit working"

# Test MySQL (replace SITE_ID)
~/.config/Local/lightning-services/mysql-*/bin/linux/bin/mysql \
  --socket=~/.config/Local/run/$SITE_ID/mysql/mysqld.sock \
  -u root -e "SELECT 'MySQL working!' as status;"

# Check Nginx capabilities
/usr/sbin/getcap ~/.config/Local/lightning-services/nginx-*/bin/linux/sbin/nginx
```

## Creating a Startup Script

For convenience, create a script to launch LocalWP with the correct parameters:

```bash
# Create startup script
cat > ~/start-localwp.sh << 'EOF'
#!/bin/bash
echo "Starting LocalWP for Crostini..."
cd /opt/Local
./local --no-sandbox --disable-gpu
EOF

# Make executable
chmod +x ~/start-localwp.sh

# Launch with:
~/start-localwp.sh
```

## Maintenance Notes

### After LocalWP Updates

1. **Re-apply Nginx capabilities:**
   ```bash
   sudo setcap CAP_NET_BIND_SERVICE=+eip ~/.config/Local/lightning-services/nginx-*/bin/linux/sbin/nginx
   ```

2. **Verify polkit rules are still active:**
   ```bash
   pkexec echo "Testing polkit after update"
   ```

### Creating New Sites

When creating new WordPress sites:

1. Use the LocalWP GUI normally
2. If MySQL fails, repeat Step 7 with the new site's directory
3. If other services fail, repeat Step 8 for PHP-FPM

## Performance Tips

1. **Always use the recommended launch flags:** `--no-sandbox --disable-gpu`
2. **Close other heavy applications** when running LocalWP in Crostini
3. **Consider increasing Crostini's allocated resources** in Chrome OS settings if available

## Known Limitations

- GPU acceleration is disabled (required for Crostini compatibility)
- Sandbox mode is disabled (required for Crostini compatibility)
- Some LocalWP features that require deep system integration may not work
- Performance may be slower than native Linux installations

## Success Indicators

When everything is working correctly, you should see:

- ✅ LocalWP launches without authentication prompts
- ✅ Sites can be created successfully
- ✅ MySQL starts and accepts connections
- ✅ WordPress sites are accessible via browser
- ✅ All services (MySQL, PHP, Nginx) show as running in LocalWP

## Additional Resources

- [LocalWP Official Documentation](https://localwp.com/help-docs/)
- [Chrome OS Linux Setup Guide](https://support.google.com/chromebook/answer/9145439)
- [Polkit Documentation](https://www.freedesktop.org/software/polkit/docs/latest/)

## Contributing

If you encounter additional issues or find improvements to this guide, please consider contributing back to help other developers.

---

**Author's Note:** This guide was developed through extensive testing and troubleshooting in a real Crostini environment. All steps have been verified to work as of LocalWP version 9.2.4.

**Last Updated:** June 8, 2025
**LocalWP Version Tested:** 9.2.4+6788
**Crostini Environment:** Debian 12 (bookworm)