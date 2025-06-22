# Hardened Dogecoin Node

A secure, production-ready Dogecoin node setup with systemd service configuration and comprehensive security hardening measures.

## 🚀 Features

- **Security Hardened**: Implements multiple security measures including:
  - Private `/tmp` and `/var/tmp` directories
  - Read-only system directories (`/usr`, `/boot`, `/etc`)
  - Protected home directories
  - No new privileges execution
  - Private device namespace
  - Memory write/execute protection
- **Systemd Integration**: Runs as a systemd service with automatic startup
- **Dedicated User**: Runs under a dedicated `dogeuser` account for security
- **Production Ready**: Configured for high-performance node operation
- **Easy Management**: Simple start/stop/status commands

## 📋 Prerequisites

- Linux system with systemd
- Dogecoin Core 1.14.6 installed in `/opt/dogecoin-1.14.6/`
- At least 50GB of free disk space for the blockchain
- Stable internet connection for initial sync

## 🛠️ Installation

### 1. Create Configuration Directory
```bash
sudo mkdir /etc/dogecoin/
```

### 2. Configure Dogecoin
Copy the provided configuration file:
```bash
sudo cp dogecoin.conf /etc/dogecoin/dogecoin.conf
```

**Important**: Edit the configuration file to set your own RPC password:
```bash
sudo nano /etc/dogecoin/dogecoin.conf
```

Update the `rpcpassword` field with a strong, unique password.

### 3. Install Systemd Service
Copy the service file to the systemd directory:
```bash
sudo cp dogecoind.service /etc/systemd/system/dogecoind.service
```

### 4. Create Dedicated User and Group
```bash
# Create dogecoin user (disabled password, no home directory)
sudo adduser --disabled-password --gecos "" dogeuser

# Create dogecoin group
sudo addgroup dogegroup

# Add user to group
sudo usermod -aG dogegroup dogeuser
```

### 5. Create Data Directory
```bash
# Create data directory (adjust path as needed)
sudo mkdir -p /mnt/data/doge
sudo chown dogeuser:dogegroup /mnt/data/doge
```

### 6. Set Permissions
```bash
# Set proper permissions for configuration
sudo chown root:dogegroup /etc/dogecoin/dogecoin.conf
sudo chmod 640 /etc/dogecoin/dogecoin.conf
```

### 7. Enable and Start Service
```bash
# Reload systemd daemon
sudo systemctl daemon-reload

# Start the service
sudo systemctl start dogecoind

# Enable service to start on boot
sudo systemctl enable dogecoind
```

## 🎮 Usage

### Service Management
```bash
# Check service status
sudo systemctl status dogecoind

# Start the service
sudo systemctl start dogecoind

# Stop the service
sudo systemctl stop dogecoind

# Restart the service
sudo systemctl restart dogecoind

# View logs
sudo journalctl -u dogecoind -f
```

### Configuration
The main configuration file is located at `/etc/dogecoin/dogecoin.conf`. Key settings include:

- `rpcuser` and `rpcpassword`: For RPC access
- `dbcache=1024`: Database cache size in MB
- `maxmempool=500`: Maximum memory pool size in MB
- `txindex=1`: Enable transaction index
- `datadir=/mnt/data/doge/`: Blockchain data directory

## 🔒 Security Features

This setup implements several security hardening measures:

- **Process Isolation**: Runs in a private namespace with restricted access
- **File System Protection**: System directories are mounted read-only
- **Memory Protection**: Prevents executable memory mappings
- **Privilege Reduction**: Runs as non-root user with minimal privileges
- **Network Security**: RPC access requires authentication

## 📁 File Structure

```
/etc/dogecoin/
├── dogecoin.conf          # Main configuration file

/etc/systemd/system/
└── dogecoind.service      # Systemd service definition

/mnt/data/doge/            # Blockchain data directory (configurable)
```

## 🔧 Troubleshooting

### Common Issues

1. **Service fails to start**
   - Check logs: `sudo journalctl -u dogecoind -n 50`
   - Verify data directory permissions
   - Ensure Dogecoin Core is installed in `/opt/dogecoin-1.14.6/`

2. **Permission denied errors**
   - Verify user/group setup: `id dogeuser`
   - Check file permissions: `ls -la /etc/dogecoin/`

3. **RPC connection issues**
   - Verify RPC credentials in configuration
   - Check firewall settings if accessing remotely

### Log Locations
- Systemd logs: `sudo journalctl -u dogecoind`
- Dogecoin debug log: `/mnt/data/doge/debug.log`

## 📊 Monitoring

Monitor your node's health:
```bash
# Check sync status
dogecoin-cli -conf=/etc/dogecoin/dogecoin.conf getblockchaininfo

# Check network connections
dogecoin-cli -conf=/etc/dogecoin/dogecoin.conf getconnectioncount

# Check memory usage
dogecoin-cli -conf=/etc/dogecoin/dogecoin.conf getmempoolinfo
```

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## 📄 License

This project is licensed under the GNU General Public License v3.0 - see the [LICENSE](LICENSE) file for details.

## ⚠️ Disclaimer

This setup is designed for production use but should be thoroughly tested in your environment before deployment. Always backup your configuration and wallet files.

## 🔗 Resources

- [Dogecoin Core Documentation](https://github.com/dogecoin/dogecoin)
- [Systemd Service Documentation](https://www.freedesktop.org/software/systemd/man/systemd.service.html)
- [Dogecoin Network](https://dogecoin.com/)
