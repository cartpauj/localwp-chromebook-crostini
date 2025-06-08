# LocalWP Crostini Quick Setup

## TL;DR - Fast Setup for Experienced Developers

**⚠️ CRITICAL:** Configure polkit BEFORE installing LocalWP to prevent configuration issues!

```bash
# 1. Install dependencies
sudo apt update && sudo apt install -y wget dpkg libaio1 libncurses5 libnss3-tools policykit-1 policykit-1-gnome libcap2-bin

# 2. Create polkit rules FIRST (CRITICAL - Replace YOUR_USERNAME)
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

# 3. Reload polkit
sudo pkill -HUP polkitd

# 4. Test polkit (MUST work without password before proceeding!)
pkexec echo "Polkit test"

# 5. Download and install LocalWP (AFTER polkit is working)
wget -O local-linux.deb "https://cdn.localwp.com/stable/latest/deb"
sudo dpkg -i local-linux.deb
sudo apt --fix-broken install -y

# 6. Launch LocalWP (should work completely now)
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
- **Critical:** Replace `YOUR_USERNAME` in polkit rules with actual username (`echo $USER`)
- **Test polkit:** MUST work without password before proceeding to LocalWP installation
- **After updates:** Re-run nginx setcap command

## Why This Order Matters

**Polkit First = Success:** Configuring polkit before LocalWP installation prevents the site creation process from failing partway through, which leaves broken configuration files.

**Wrong Order = Problems:** Installing LocalWP first leads to incomplete site creation and requires manual configuration fixes.

## Verification

```bash
pkexec echo "Polkit OK"  # Should work without password
/usr/sbin/getcap ~/.config/Local/lightning-services/nginx-*/bin/linux/sbin/nginx  # Should show cap_net_bind_service
```

Ready to rock! 🚀