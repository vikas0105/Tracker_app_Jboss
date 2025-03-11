#!/bin/bash

# Variables
JBOSS_VERSION="26.1.1.Final"
JBOSS_USER="jboss"
JBOSS_GROUP="jboss"
INSTALL_DIR="/opt/jboss"
DOWNLOAD_URL="https://github.com/wildfly/wildfly/releases/download/$JBOSS_VERSION/wildfly-$JBOSS_VERSION.tar.gz"
SERVICE_FILE="/etc/systemd/system/jboss.service"

# Ensure script is run as root
if [[ $EUID -ne 0 ]]; then
    echo "❌ This script must be run as root" >&2
    exit 1
fi

# Stop JBoss service if running
if systemctl is-active --quiet jboss; then
    echo "🛑 Stopping JBoss service..."
    systemctl stop jboss
fi

# Remove old JBoss versions
if [[ -d "$INSTALL_DIR" ]]; then
    echo "🧹 Removing previous JBoss installation..."
    rm -rf "$INSTALL_DIR"
fi

# Remove old systemd service file
if [[ -f "$SERVICE_FILE" ]]; then
    echo "🧹 Removing old systemd service file..."
    rm -f "$SERVICE_FILE"
    systemctl daemon-reload
fi

# Install dependencies
echo "📦 Updating system and installing Java..."
yum update -y
yum install -y java-11-amazon-corretto wget tar

# Create JBoss user and group if they don't exist
if ! id "$JBOSS_USER" &>/dev/null; then
    echo "👤 Creating JBoss user..."
    useradd -r -m -d "$INSTALL_DIR" -s /sbin/nologin "$JBOSS_USER"
fi

# Download and install JBoss
echo "⬇️ Downloading JBoss WildFly..."
cd /opt
wget "$DOWNLOAD_URL" -O wildfly.tar.gz

if [[ ! -f wildfly.tar.gz ]]; then
    echo "❌ Error: JBoss download failed!"
    exit 1
fi

echo "📂 Extracting JBoss..."
tar -xvzf wildfly.tar.gz
rm -f wildfly.tar.gz
mv wildfly-$JBOSS_VERSION jboss
chown -R $JBOSS_USER:$JBOSS_GROUP $INSTALL_DIR
chmod -R 755 $INSTALL_DIR/bin

# Configure systemd service
echo "⚙️ Configuring systemd service..."
cat > "$SERVICE_FILE" <<EOF
[Unit]
Description=JBoss WildFly Application Server
After=network.target

[Service]
User=$JBOSS_USER
Group=$JBOSS_GROUP
ExecStart=/bin/bash $INSTALL_DIR/bin/standalone.sh -b 0.0.0.0
ExecStop=$INSTALL_DIR/bin/jboss-cli.sh --connect command=:shutdown
Restart=always
LimitNOFILE=102642
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd, enable and start JBoss
chmod 644 "$SERVICE_FILE"
systemctl daemon-reload
systemctl enable jboss
systemctl start jboss

# Verify if JBoss is running
echo "🔍 Verifying JBoss service status..."
if systemctl is-active --quiet jboss; then
    echo "✅ JBoss successfully installed and running!"
    echo "🌍 Access it at: http://$(curl -s ifconfig.me):8080"
else
    echo "❌ JBoss service failed to start. Check logs:"
    journalctl -u jboss --no-pager --lines=50
fi
========================================================================================================================================================================================================================================================================================================================================================================================================================================
