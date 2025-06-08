# LocalWP Crostini Troubleshooting Reference

## ⚠️ Most Important: Installation Order

**If you're experiencing multiple configuration issues (MySQL, PHP-FPM, Nginx), the root cause is likely that polkit authentication wasn't configured before installing LocalWP.**

**Quick Fix:**
1. **Uninstall LocalWP:** `sudo apt remove local -y && sudo rm -rf /opt/Local && rm -rf ~/.config/Local`
2. **Follow the updated setup guide** which configures polkit FIRST
3. **Reinstall with correct order** - this should prevent most issues

## Common Error Messages and Solutions

### 1. "No polkit authentication agent found"

**Error in logs:**
```
Error: No polkit authentication agent found.
Error executing command as another user: No authentication agent found.
exitCode: 127
```

**Root Cause:** Crostini doesn't have a proper polkit authentication agent for GUI applications.

**Solution:**
1. Install polkit components: `sudo apt install -y policykit-1 policykit-1-gnome`
2. Create polkit rules (see main guide Step 4)
3. Reload polkit: `sudo pkill -HUP polkitd`
4. Test: `pkexec echo "test"` (should work without password)

### 2. MySQL Connection Failures

**Error in logs:**
```
Database connection attempt X failed. Retrying in Xms.
mysqladmin: connect to server at 'localhost' failed
error: 'Can't connect to local MySQL server through socket '/tmp/mysql.sock' (2)'
```

**Root Cause:** MySQL configuration files weren't generated due to template processing failure, usually caused by polkit authentication errors interrupting site creation.

**Prevention:** Configure polkit authentication BEFORE installing LocalWP (see updated setup guides).

**Solution (if already encountered):**
1. Create MySQL config directory: `mkdir -p ~/.config/Local/run/SITE_ID/conf/mysql`
2. Create `my.cnf` file (see main guide Step 7.2)
3. Initialize database: Use mysqld --initialize-insecure (see main guide Step 7.3)

### 3. "Unable to find pkexec or kdesudo"

**Error in logs:**
```
Error: Unable to find pkexec or kdesudo.
```

**Root Cause:** Missing privilege escalation tools.

**Solution:**
```bash
sudo apt install -y policykit-1 pkexec
```

### 4. PHP-FPM Configuration Errors

**Error in logs:**
```
ERROR: failed to open configuration file 'site.runData/conf/php/php-fpm.conf': No such file or directory (2)
ERROR: FPM initialization failed
```

**Root Cause:** PHP-FPM configuration template wasn't processed.

**Solution:**
```bash
mkdir -p ~/.config/Local/run/SITE_ID/conf/php
cat > ~/.config/Local/run/SITE_ID/conf/php/php-fpm.conf << EOF
[global]
error_log = "/home/$USER/.config/Local/run/SITE_ID/logs/php-fpm.log"
daemonize = no
include=php-fpm.d/*.conf
EOF
```

### 5. Nginx Permission Denied

**Error in logs:**
```
nginx: [alert] could not open error log file
open() "site.runData/conf/nginx/nginx.conf" failed (2: No such file or directory)
```

**Root Cause:** Either missing configuration files or insufficient privileges.

**Solutions:**
1. **For privilege issues:** `sudo setcap CAP_NET_BIND_SERVICE=+eip ~/.config/Local/lightning-services/nginx-*/bin/linux/sbin/nginx`
2. **For missing config:** Usually resolves when MySQL is fixed and site creation completes

### 6. "Fatal error in defaults handling"

**Error in logs:**
```
mysqld: [ERROR] Fatal error in defaults handling. Program aborted!
```

**Root Cause:** MySQL can't find or parse the configuration file.

**Solution:**
1. Verify `my.cnf` exists and has correct paths
2. Check file permissions: `ls -la ~/.config/Local/run/SITE_ID/conf/mysql/my.cnf`
3. Recreate config file with absolute paths

### 7. GUI Doesn't Appear / Crashes

**Error:** LocalWP starts but no window appears, or immediate crashes.

**Root Cause:** GPU/sandbox issues in Crostini environment.

**Solution:**
Always launch with: `/opt/Local/local --no-sandbox --disable-gpu`

### 8. "Cannot create unix session: No session for pid"

**Error in logs:**
```
Cannot create unix session: No session for pid XXXX
Cannot register authentication agent!
```

**Root Cause:** Crostini session management limitations.

**Solution:**
This is expected in Crostini. The polkit rules we create bypass this requirement.

## Debugging Commands

### Check System Status

```bash
# Check if polkit daemon is running
sudo systemctl status polkit

# Check current polkit rules
sudo ls -la /etc/polkit-1/rules.d/

# Test polkit without password
pkexec echo "Polkit test"

# Check LocalWP processes
ps aux | grep -i local

# Check MySQL processes
ps aux | grep mysqld
```

### Check LocalWP Installation

```bash
# Verify LocalWP is installed
which local
/opt/Local/local --version

# Check dependencies
dpkg -l | grep -E "(libaio1|libncurses5|libnss3-tools|policykit)"

# Check nginx capabilities
/usr/sbin/getcap ~/.config/Local/lightning-services/nginx-*/bin/linux/sbin/nginx
```

### Check Site Configuration

```bash
# List LocalWP sites
ls ~/.config/Local/run/

# Check specific site config (replace SITE_ID)
ls -la ~/.config/Local/run/SITE_ID/conf/

# Test MySQL connection manually
~/.config/Local/lightning-services/mysql-*/bin/linux/bin/mysql \
  --socket=~/.config/Local/run/SITE_ID/mysql/mysqld.sock \
  -u root -e "SELECT 'Working!' as status;"
```

## Log Locations

```bash
# LocalWP logs (when running from terminal)
# Shown in terminal output with timestamps

# System polkit logs
sudo journalctl -u polkit -n 20

# MySQL error logs (if configured)
~/.config/Local/run/SITE_ID/logs/mysql-error.log

# PHP-FPM logs (if configured)  
~/.config/Local/run/SITE_ID/logs/php-fpm.log
```

## Environment Verification

### Prerequisites Check

```bash
# Check Crostini environment
echo $XDG_SESSION_TYPE  # Should show 'wayland'
uname -a  # Should show Linux with ChromeOS kernel

# Check user permissions
id  # Should show user in sudo group
groups | grep sudo  # Should include sudo

# Check polkit installation
which pkexec  # Should show /usr/bin/pkexec
systemctl status polkit  # Should show active
```

### Post-Installation Verification

```bash
# Test privilege escalation
pkexec echo "Success"  # Should work without password

# Test LocalWP components
which local  # Should show /usr/bin/local
ls /opt/Local/  # Should show LocalWP files

# Test capabilities
/usr/sbin/getcap ~/.config/Local/lightning-services/nginx-*/bin/linux/sbin/nginx
# Should show: cap_net_bind_service=eip
```

## Recovery Procedures

### Complete Reset

If LocalWP is completely broken:

```bash
# Remove LocalWP
sudo apt remove local -y
sudo rm -rf /opt/Local
rm -rf ~/.config/Local

# Remove polkit rules
sudo rm -f /etc/polkit-1/rules.d/99-localwp-allow.rules
sudo pkill -HUP polkitd

# Start fresh with installation guide
```

### Site-Specific Reset

If a specific site is broken:

```bash
# Find problematic site
ls ~/.config/Local/run/

# Remove site data (replace SITE_ID)
rm -rf ~/.config/Local/run/SITE_ID

# Recreate site through LocalWP GUI
```

## Performance Optimization

### If LocalWP is slow:

1. **Close other Chrome OS apps** to free memory
2. **Increase Crostini resources** in Chrome OS settings
3. **Use SSD storage** for better I/O performance
4. **Disable unnecessary LocalWP features** in preferences

### Memory Issues:

```bash
# Check available memory
free -h

# Check LocalWP memory usage
ps aux | grep local | awk '{print $6}' | head -1
```

## Getting Help

1. **Check this troubleshooting guide first**
2. **Run verification commands** to identify the specific issue
3. **Collect logs** from terminal output when running LocalWP
4. **Test individual components** (polkit, MySQL, nginx) separately
5. **Try the complete reset** if all else fails

Remember: Most issues in Crostini stem from privilege escalation problems or missing configuration files due to template processing failures.