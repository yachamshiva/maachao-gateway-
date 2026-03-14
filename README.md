# Maachao High-Frequency Gateway

**Author:** Yacham Sivakumar  
**Email:** sivakumar05426@gmail.com  
**Date:** March 2026  
**Platform:** CentOS 7 / Linux

---

## 🎯 Project Overview

A production-ready security infrastructure for high-frequency, real-time networking servers. This implementation includes:

- **Strict Firewall Configuration** (firewalld/ufw)
- **Nginx Reverse Proxy** with rate limiting (5 req/s per IP)
- **Node.js Backend API** (completely protected from direct access)
- **Automated Defense System** (auto-bans IPs violating rate limits)

Built to protect against automated botnets and DDoS attacks while supporting high-frequency real-time UDP streaming.

---

## 📋 Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Testing](#testing)
- [Security](#security)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

---

## ✨ Features

### 🛡️ Multi-Layered Security
- Firewall with strict deny-all-by-default policy
- Custom SSH port (2222) to reduce bot attacks
- Port 3000 explicitly blocked from external access
- Automated IP banning for rate limit violators

### 🚀 High Performance
- Nginx reverse proxy with efficient request routing
- Rate limiting: 5 requests/second per IP with burst handling
- Node.js backend optimized for high-frequency requests
- Support for UDP streaming (ports 10000-20000)

### 🤖 Automated Defense
- Real-time log parsing for 429 errors
- Automatic IP banning after 3 violations
- Persistent banlist with audit logging
- Cron-based continuous monitoring (every 1 minute)

### 📊 Production Ready
- PM2 process management with auto-restart
- Comprehensive logging and error handling
- SELinux properly configured (CentOS)
- Graceful shutdown handling

---

## 🏗️ Architecture

```
                                    INTERNET
                                        │
                                        │
                         ┌──────────────▼──────────────┐
                         │   Firewall (firewalld)      │
                         │  ┌─────────────────────┐    │
                         │  │ Port 2222 (SSH)     │    │
                         │  │ Port 80 (HTTP)      │    │
                         │  │ Port 443 (HTTPS)    │    │
                         │  │ Port 10000-20000    │    │
                         │  │      (UDP)          │    │
                         │  │ Port 3000 BLOCKED   │    │
                         │  └─────────────────────┘    │
                         └──────────────┬──────────────┘
                                        │
                         ┌──────────────▼──────────────┐
                         │   Nginx Reverse Proxy       │
                         │  ┌─────────────────────┐    │
                         │  │ Rate Limiting:      │    │
                         │  │ 5 req/s per IP      │    │
                         │  │ Returns 429 on      │    │
                         │  │ violation           │    │
                         │  └─────────────────────┘    │
                         └──────────────┬──────────────┘
                                        │
                         ┌──────────────▼──────────────┐
                         │   Node.js Backend           │
                         │   (localhost:3000)          │
                         │  ┌─────────────────────┐    │
                         │  │ Express.js API      │    │
                         │  │ PM2 Managed         │    │
                         │  │ Auto-restart        │    │
                         │  └─────────────────────┘    │
                         └─────────────────────────────┘
                                        
         ┌───────────────────────────────────────────────┐
         │   Defense Script (defender.sh)                │
         │  ┌──────────────────────────────────────┐     │
         │  │ 1. Scan Nginx logs for 429 errors   │     │
         │  │ 2. Count violations per IP           │     │
         │  │ 3. Ban IPs with 3+ violations        │     │
         │  │ 4. Update firewall + banlist         │     │
         │  │ 5. Send alerts                       │     │
         │  └──────────────────────────────────────┘     │
         │  Runs every 1 minute via cron                 │
         └───────────────────────────────────────────────┘
```

---

## 🚀 Installation

### Prerequisites

**CentOS 7:**
```bash
sudo yum update -y
sudo yum install -y epel-release vim curl wget git net-tools policycoreutils-python
```

**Ubuntu 20.04/22.04:**
```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y vim curl wget git ufw
```

### Quick Install Script

```bash
# Clone repository
git clone https://github.com/yachamshiva/maachao-gateway.git
cd maachao-gateway

# Run setup script
chmod +x scripts/setup.sh
sudo ./scripts/setup.sh
```

### Manual Installation

#### Step 1: Install Node.js

```bash
# Install Node.js 20.x LTS
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -  # CentOS
# OR
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -  # Ubuntu

# Install Node.js
sudo yum install -y nodejs  # CentOS
# OR
sudo apt install -y nodejs  # Ubuntu

# Verify
node --version
npm --version
```

#### Step 2: Install Nginx

```bash
# CentOS
sudo yum install -y nginx

# Ubuntu
sudo apt install -y nginx

# Start and enable
sudo systemctl start nginx
sudo systemctl enable nginx
```

#### Step 3: Configure Firewall

**CentOS (firewalld):**
```bash
sudo systemctl start firewalld
sudo systemctl enable firewalld

# Configure rules
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --permanent --add-port=10000-20000/udp
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" port port="3000" protocol="tcp" reject'
sudo firewall-cmd --reload
```

**Ubuntu (ufw):**
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp
sudo ufw allow 10000:20000/udp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw deny 3000/tcp
sudo ufw enable
```

#### Step 4: Configure SELinux (CentOS only)

```bash
sudo setsebool -P httpd_can_network_connect 1
sudo semanage port -a -t ssh_port_t -p tcp 2222
sudo semanage port -a -t http_port_t -p tcp 3000
```

#### Step 5: Deploy Application

```bash
# Copy Nginx config
sudo cp nginx/maachao-gateway.conf /etc/nginx/conf.d/  # CentOS
# OR
sudo cp nginx/maachao-gateway.conf /etc/nginx/sites-available/
sudo ln -s /etc/nginx/sites-available/maachao-gateway /etc/nginx/sites-enabled/  # Ubuntu

# Test and reload Nginx
sudo nginx -t
sudo systemctl reload nginx

# Deploy Node.js backend
sudo mkdir -p /opt/maachao-gateway
sudo cp backend/server.js /opt/maachao-gateway/
sudo cp backend/package.json /opt/maachao-gateway/
sudo cp backend/ecosystem.config.js /opt/maachao-gateway/
cd /opt/maachao-gateway
npm install

# Install and configure PM2
sudo npm install -g pm2
pm2 start ecosystem.config.js
pm2 save
pm2 startup systemd

# Deploy defense script
sudo mkdir -p /opt/maachao-defense
sudo cp defense/defender.sh /opt/maachao-defense/
chmod +x /opt/maachao-defense/defender.sh

# Setup cron
crontab -e
# Add: * * * * * /opt/maachao-defense/defender.sh
```

---

## ⚙️ Configuration

### Nginx Configuration

Located at: `nginx/maachao-gateway.conf`

Key settings:
```nginx
# Rate limiting zone
limit_req_zone $binary_remote_addr zone=maachao_limit:10m rate=5r/s;

# In server block
limit_req zone=maachao_limit burst=10 nodelay;
limit_req_status 429;
```

### Node.js Configuration

Located at: `backend/server.js`

Main endpoint:
```javascript
app.get('/api/status', (req, res) => {
  res.json({
    status: 'secure',
    message: 'Maachao Gateway Active'
  });
});
```

### Defense Script Configuration

Located at: `defense/defender.sh`

Configurable variables:
```bash
VIOLATION_THRESHOLD=3        # Number of violations before ban
NGINX_ERROR_LOG="/var/log/nginx/maachao_error.log"
BANLIST_FILE="/opt/maachao-defense/banned_ips.txt"
```

---

## 📖 Usage

### Testing Rate Limiting

```bash
# Single request (should succeed)
curl http://YOUR_SERVER_IP/api/status

# Rapid requests (will trigger 429)
for i in {1..20}; do 
    curl -w "Status: %{http_code}\n" http://YOUR_SERVER_IP/api/status
done
```

### Checking Firewall Rules

**CentOS:**
```bash
sudo firewall-cmd --list-all
sudo firewall-cmd --list-rich-rules
```

**Ubuntu:**
```bash
sudo ufw status numbered
sudo ufw status verbose
```

### Viewing Logs

```bash
# Nginx access log
sudo tail -f /var/log/nginx/maachao_access.log

# Nginx error log
sudo tail -f /var/log/nginx/maachao_error.log

# Defense script log
sudo tail -f /var/log/maachao-defense/defender.log

# Node.js application log
pm2 logs maachao-gateway
```

### Manual Defense Script Execution

```bash
cd /opt/maachao-defense
./defender.sh
```

### Checking Banned IPs

```bash
# View banlist
cat /opt/maachao-defense/banned_ips.txt

# View firewall bans (CentOS)
sudo firewall-cmd --list-rich-rules | grep drop

# View firewall bans (Ubuntu)
sudo ufw status | grep DENY
```

---

## 🧪 Testing

### Test Suite

```bash
# Run all tests
cd tests
./run_tests.sh

# Individual tests
./test-firewall.sh       # Test firewall rules
./test-rate-limit.sh     # Test rate limiting
./test-backend.sh        # Test backend API
./test-defense.sh        # Test defense script
```

### Manual Testing

1. **Test Backend Protected:**
   ```bash
   curl http://YOUR_SERVER_IP:3000/api/status  # Should fail
   ```

2. **Test Nginx Proxy Works:**
   ```bash
   curl http://YOUR_SERVER_IP/api/status  # Should succeed
   ```

3. **Test Rate Limiting:**
   ```bash
   ab -n 100 -c 10 http://YOUR_SERVER_IP/api/status
   # Or use: tests/rate-limit-test.py
   ```

4. **Test Defense Script:**
   ```bash
   # Trigger violations
   for i in {1..50}; do curl http://YOUR_SERVER_IP/api/status; done
   
   # Run script
   ./defender.sh
   
   # Verify ban
   sudo firewall-cmd --list-rich-rules | grep YOUR_IP
   ```

---

## 🔒 Security

### Security Features

- ✅ Firewall with default deny policy
- ✅ SSH on non-standard port (2222)
- ✅ Backend completely isolated from internet
- ✅ Rate limiting to prevent abuse
- ✅ Automated IP banning
- ✅ SELinux enforcing (CentOS)
- ✅ Security headers in Nginx
- ✅ Audit logging

### Security Best Practices

1. **Change SSH Keys:**
   ```bash
   ssh-keygen -t ed25519
   ssh-copy-id -p 2222 user@server
   ```

2. **Disable Password Authentication:**
   ```bash
   # In /etc/ssh/sshd_config
   PasswordAuthentication no
   ```

3. **Keep System Updated:**
   ```bash
   sudo yum update -y  # CentOS
   sudo apt update && sudo apt upgrade -y  # Ubuntu
   ```

4. **Monitor Logs Regularly:**
   ```bash
   sudo tail -f /var/log/secure  # SSH attempts
   sudo tail -f /var/log/nginx/maachao_error.log
   ```

---

## 📊 Monitoring

### Real-time Monitoring

```bash
# Watch defense log
watch -n 5 'tail -20 /var/log/maachao-defense/defender.log'

# Monitor Nginx access
tail -f /var/log/nginx/maachao_access.log | grep 429

# Monitor system resources
htop
```

### Statistics

```bash
# Total banned IPs
wc -l /opt/maachao-defense/banned_ips.txt

# Recent 429 errors
grep "429" /var/log/nginx/maachao_access.log | wc -l

# Application uptime
pm2 status
```

---

## 🐛 Troubleshooting

### Common Issues

**1. Nginx 502 Bad Gateway**
```bash
# Check if Node.js is running
pm2 status

# Check if listening on port 3000
sudo netstat -tulpn | grep 3000

# Check SELinux (CentOS)
sudo setsebool -P httpd_can_network_connect 1
```

**2. Can't SSH After Port Change**
```bash
# From cloud console or KVM, revert SSH config
sudo vim /etc/ssh/sshd_config
# Change Port back to 22
sudo systemctl restart sshd
```

**3. Defense Script Not Banning**
```bash
# Check sudo permissions
sudo visudo
# Add: username ALL=(ALL) NOPASSWD: /usr/bin/firewall-cmd

# Or check script has execute permission
chmod +x /opt/maachao-defense/defender.sh
```

**4. Rate Limiting Not Working**
```bash
# Test Nginx config
sudo nginx -t

# Check if zone is defined
grep -A 5 "limit_req_zone" /etc/nginx/conf.d/maachao-gateway.conf

# Reload Nginx
sudo systemctl reload nginx
```

---

## 🤝 Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -am 'Add improvement'`)
4. Push to the branch (`git push origin feature/improvement`)
5. Create a Pull Request

---

## 📄 License

MIT License - see [LICENSE](LICENSE) file for details

---

## 👤 Author

**Yacham Sivakumar**
- Email: sivakumar05426@gmail.com
- GitHub: [@yachamshiva](https://github.com/yachamshiva)
- LinkedIn: [Yacham Sivakumar](https://linkedin.com/in/yachamshiva)

---

## 🙏 Acknowledgments

- Maachao for the challenging assignment
- The Nginx and Node.js communities
- Linux security best practices documentation

---

## 📚 Additional Resources

- [Nginx Rate Limiting Guide](https://www.nginx.com/blog/rate-limiting-nginx/)
- [firewalld Documentation](https://firewalld.org/documentation/)
- [PM2 Process Manager](https://pm2.keymetrics.io/)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)

---

**Last Updated:** March 2026  
**Version:** 1.0.0  
**Status:** Production Ready ✅
