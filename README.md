# LocalWP for Chrome OS Crostini

## Requirements
- Must use the primary penguin Linux container "penguin"

## Setup Steps

### 1. Find Your Username
```bash
echo $USER
# or
whoami
# or
users
```

### 2. Fix Polkit Authentication
Replace `YOUR_USERNAME` with your actual username from step 1:

```bash
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

sudo pkill -HUP polkitd
```

### 3. Restart Linux Environment
Either restart your Chromebook or right-click the Linux icon in the shelf and choose "Shutdown Linux".

### 4. Install LocalWP
Download and install LocalWP after the restart.

### 5. Configure Router Mode
In LocalWP Preferences, set Router Mode to "localhost".

Your LocalWP sites will now be accessible from Chrome OS browser via `http://localhost:PORT/`