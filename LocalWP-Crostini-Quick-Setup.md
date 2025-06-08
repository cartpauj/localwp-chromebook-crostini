# LocalWP Crostini Quick Setup

## TL;DR - Fast Setup for Experienced Developers

**⚠️ CRITICAL:** Configure polkit BEFORE installing LocalWP to prevent configuration issues!

```bash
# 1. Install dependencies
sudo apt update && sudo apt install -y wget dpkg libaio1 libncurses5 libnss3-tools policykit-1 policykit-1-gnome libcap2-bin

# 2. Find your username (IMPORTANT - Don't skip this!)
echo "Your username is: $USER"
# You can also use: whoami
# Or check all users: users

# 3. Create polkit rules FIRST (CRITICAL - Replace YOUR_USERNAME with actual username)
sudo tee /etc/polkit-1/rules.d/99-localwp-allow.rules > /dev/null << EOF
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.policykit.exec" && subject.user == "YOUR_USERNAME") {
        return polkit.Result.YES;
    }
});
polkit.addRule(function(action, subject) {
    if (action.id.indexOf("org.freedesktop") == 0 && subject.user == "YOUR_USERNAME") {
        return polkit.Result.YES;
    }
});
polkit.addRule(function(action, subject) {
    if (subject.user == "YOUR_USERNAME" && subject.local) {
        return polkit.Result.YES;
    }
});
EOF

# 4. Reload polkit
sudo pkill -HUP polkitd

# 5. Test polkit (MUST work without password before proceeding!)
pkexec echo "Polkit test"

# 6. IMPORTANT: Restart Crostini container (fixes any lingering issues)
# In Chrome OS: Settings > Advanced > Developers > Linux development environment > "Stop" then "Turn on"
# Or from terminal: sudo shutdown -h now
# Then restart Linux from Chrome OS settings
echo "⚠️  RESTART YOUR CROSTINI CONTAINER NOW before proceeding!"
echo "This clears system state and ensures LocalWP works correctly."

# 7. Download and install LocalWP (AFTER container restart)
wget -O local-linux.deb "https://cdn.localwp.com/stable/latest/deb"
sudo dpkg -i local-linux.deb
sudo apt --fix-broken install -y

# 8. Launch LocalWP (should work completely now)
/opt/Local/local --no-sandbox --disable-gpu
```

## MySQL Fix (likely NOT needed with correct order!)

**Note:** With polkit configured first, you probably won't need this section. Only use if MySQL still fails:

```bash
# Find your site ID
SITE_ID=$(ls ~/.config/Local/run/ | head -1)

# Create MySQL config
mkdir -p ~/.config/Local/run/$SITE_ID/conf/mysql
cat > ~/.config/Local/run/$SITE_ID/conf/mysql/my.cnf << EOF
[mysqld]
skip-name-resolve
datadir = /home/$USER/.config/Local/run/$SITE_ID/mysql/data
port = 10004
bind-address = 127.0.0.1
socket = /home/$USER/.config/Local/run/$SITE_ID/mysql/mysqld.sock
character-set-server = utf8mb3
default_authentication_plugin = mysql_native_password
performance_schema = off
max_allowed_packet = 16M
innodb_buffer_pool_size = 32M
innodb_log_file_size = 96M

[client]
socket = /home/$USER/.config/Local/run/$SITE_ID/mysql/mysqld.sock
user = root
password = root
EOF

# Initialize MySQL
mkdir -p ~/.config/Local/run/$SITE_ID/mysql/data
~/.config/Local/lightning-services/mysql-*/bin/linux/bin/mysqld \
  --defaults-file=~/.config/Local/run/$SITE_ID/conf/mysql/my.cnf \
  --initialize-insecure
```

## After Updates

```bash
# Re-apply nginx capabilities after LocalWP updates
sudo setcap CAP_NET_BIND_SERVICE=+eip ~/.config/Local/lightning-services/nginx-*/bin/linux/sbin/nginx
```

## Key Points

- **ORDER MATTERS:** Configure polkit BEFORE installing LocalWP!
- **Must use:** `--no-sandbox --disable-gpu` flags
- **Critical:** Replace `YOUR_USERNAME` in polkit rules with actual username (use `echo $USER` or `whoami`)
- **Test polkit:** MUST work without password before proceeding
- **Restart container:** Essential after polkit configuration - clears system state
- **After updates:** Re-run nginx setcap command

## Why This Order Matters

**Polkit First = Success:** Configuring polkit before LocalWP installation prevents the site creation process from failing partway through, which leaves broken configuration files.

**Wrong Order = Problems:** Installing LocalWP first leads to incomplete site creation and requires manual configuration fixes.

**Container Restart = Clean State:** Restarting the Crostini container after polkit configuration clears any lingering system state that might interfere with LocalWP operation.

## Verification

```bash
pkexec echo "Polkit OK"  # Should work without password
/usr/sbin/getcap ~/.config/Local/lightning-services/nginx-*/bin/linux/sbin/nginx  # Should show cap_net_bind_service
```

Ready to rock! 🚀