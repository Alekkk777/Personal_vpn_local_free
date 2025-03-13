# Complete Guide: Free VPN Configuration with Oracle Cloud Always Free

## 📋 Table of Contents
- [Prerequisites](#prerequisites)
- [Oracle Cloud Account Preparation](#oracle-cloud-account-preparation)
- [Compute Instance Configuration](#compute-instance-configuration)
- [Network and Security Configuration](#network-and-security-configuration)
- [OpenVPN Installation and Configuration](#openvpn-installation-and-configuration)
- [Certificate Generation](#certificate-generation)
- [Server Configuration](#server-configuration)
- [Client Configuration](#client-configuration)
- [Testing and Troubleshooting](#testing-and-troubleshooting)
- [Maintenance and Updates](#maintenance-and-updates)
- [Security Considerations](#security-considerations)

## 🎯 Prerequisites

### Minimum Recommended Hardware
- Computer with operating system:
  - macOS (version 10.15 or higher)
  - Linux (Ubuntu, Debian, Fedora)
  - Windows 10/11 with Windows Subsystem for Linux (WSL)
- Stable Internet connection (minimum 5 Mbps)
- Disk space: at least 10 GB

### Required Software
- Updated web browser (Chrome, Firefox, Safari)
- SSH Client (OpenSSH for Unix, PuTTY for Windows)
- Package manager (apt, yum, brew)
- Git (optional, but recommended)

### Accounts and Credentials
- Credit card (for verification, no charges will be made)
- Valid email address
- Phone number for verification

## 🌐 Oracle Cloud Account Preparation

### Registration Steps
1. Visit [oracle.com/cloud/free](https://www.oracle.com/cloud/free/)
2. Click "Start for Free"
3. Select "Create a Personal Account"
4. Fill out all form fields:
   - Full name
   - Email
   - Password (use a complex password)
   - Phone number
5. Verify email by clicking confirmation link
6. Complete phone verification with SMS code

### Account Configuration
- Select "Always Free" during registration
- Accept terms and conditions
- Enter credit card details for verification (no charges will be made)

## 🖥️ Compute Instance Configuration

### Virtual Machine Creation
1. Access Oracle Cloud Console
2. Navigate to "Compute" > "Instances"
3. Click "Create Instance"

#### Recommended Configurations
- **Image**: Ubuntu Server 22.04
- **Shape**: Always Free (AMD)
- **Network**: Default public network
- **Operating System**: Ubuntu 22.04
- **Storage Size**: Minimum 50 GB

### SSH Key Generation
```bash
# Generate SSH key
ssh-keygen -t rsa -b 4096 -f ~/.ssh/oracle_vpn_key

# Display public key
cat ~/.ssh/oracle_vpn_key.pub
```
- Copy public key during instance creation
- Save private key in a secure location

## 🔒 Network and Security Configuration

### Security Group Configuration
1. Open "Virtual Cloud Network"
2. Select default network
3. Add ingress rules:
   - Port 22 (SSH)
   - Port 1194 (OpenVPN)
   ```
   Protocol: UDP/TCP
   Port: 1194
   Source: 0.0.0.0/0
   ```

### Firewall Configuration
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install UFW (Uncomplicated Firewall)
sudo apt install ufw

# Configure basic rules
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 1194/udp
sudo ufw enable
```

## 🛠️ Dependencies Installation

```bash
# Install necessary packages
sudo apt install -y \
    openvpn \
    easy-rsa \
    wget \
    curl \
    net-tools \
    software-properties-common

# Verify installations
openvpn --version
easyrsa --version
```

## 🔐 Certificates and Keys Generation

### Complete Procedure
```bash
# Initialize PKI
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
./easyrsa init-pki

# Create Certificate Authority
./easyrsa build-ca nopass

# Generate server certificate
./easyrsa gen-req server nopass
./easyrsa sign-req server server

# Generate client certificate
./easyrsa gen-req client nopass
./easyrsa sign-req client client

# Generate Diffie-Hellman parameters
./easyrsa gen-dh

# Generate TLS key
openvpn --genkey secret ta.key
```

## 📝 Detailed OpenVPN Server Configuration

```bash
# Create server configuration file
sudo nano /etc/openvpn/server.conf
```

`server.conf` file content:
```
# Basic configurations
port 1194
proto udp
dev tun
user nobody
group nogroup

# Certificate paths
ca /home/ubuntu/openvpn-ca/pki/ca.crt
cert /home/ubuntu/openvpn-ca/pki/issued/server.crt
key /home/ubuntu/openvpn-ca/pki/private/server.key
dh /home/ubuntu/openvpn-ca/pki/dh.pem
tls-auth /home/ubuntu/openvpn-ca/ta.key 0

# Network configuration
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 1.1.1.1"

# Security settings
cipher AES-256-GCM
auth SHA512
tls-version-min 1.2
tls-cipher TLS-DHE-RSA-WITH-AES-256-GCM-SHA384

# Other settings
keepalive 10 120
comp-lzo
persist-key
persist-tun
status openvpn-status.log
verb 3
explicit-exit-notify 1
```

## 🚀 OpenVPN Service Startup

```bash
# Enable IP forwarding
sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sudo sysctl -p

# Configure NAT
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

# Start service
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server

# Check status
sudo systemctl status openvpn@server
```

## 📱 Client Configuration

### Client File Generation
```bash
# Create client file
cat > ~/client.ovpn << EOF
client
dev tun
proto udp
remote [PUBLIC-SERVER-IP] 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-GCM
auth SHA512
verb 3

<ca>
$(cat ~/openvpn-ca/pki/ca.crt)
</ca>

<cert>
$(cat ~/openvpn-ca/pki/issued/client.crt)
</cert>

<key>
$(cat ~/openvpn-ca/pki/private/client.key)
</key>

<tls-auth>
$(cat ~/openvpn-ca/ta.key)
</tls-auth>
key-direction 1
EOF
```

## 🔍 Testing and Troubleshooting

### Connectivity Verification
```bash
# Check open ports
sudo netstat -tuln | grep 1194

# View logs
sudo journalctl -u openvpn@server

# Client connection test
ping 10.8.0.1
```

## 🛡️ Security Considerations

### Recommendations
- Use complex passwords
- Keep system updated
- Limit SSH access
- Use two-factor authentication
- Monitor access and logs

### Best Practices
- Rotate certificates periodically
- Keep firewall always active
- Disable unnecessary access
- Use fail2ban to prevent attacks

## 🔄 Maintenance

### Periodic Updates
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Renew certificates
cd ~/openvpn-ca
./easyrsa renew client
./easyrsa renew server
```

## ⚖️ Legal Notes and Disclaimer

- Use VPN in compliance with local laws
- Do not use for illegal activities
- Respect Oracle Cloud terms of service
- Be aware of cybersecurity risks

## 📚 Additional Resources
- [OpenVPN Documentation](https://openvpn.net/community-resources/)
- [Oracle Cloud Documentation](https://docs.oracle.com/en-us/iaas/Content/home.htm)
- [Easy-RSA GitHub](https://github.com/OpenVPN/easy-rsa)

## 🤝 Contributions
Contributions and improvements are welcome! Open issues or pull requests.

```


