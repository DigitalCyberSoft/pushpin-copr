# Pushpin COPR Repository

Automated COPR builds of [Pushpin](https://pushpin.org/) realtime reverse proxy from GitHub releases.

## About Pushpin

Pushpin is a reverse proxy server written in Rust/C++ that makes it easy to implement WebSocket, HTTP streaming, and HTTP long-polling services. It's maintained by [Fastly](https://github.com/fastly/pushpin).

## Installation

### Enable COPR Repository

```bash
# Replace YOUR_USER with the actual COPR username
sudo dnf copr enable YOUR_USER/pushpin
sudo dnf install pushpin
```

### Manual Installation from SRPM

```bash
# Build locally
cd .copr
make srpm

# Install the generated SRPM
sudo dnf builddep *.src.rpm
rpmbuild --rebuild *.src.rpm
sudo dnf install ~/rpmbuild/RPMS/x86_64/pushpin-*.rpm
```

## Configuration

### Basic Setup

1. Edit the main configuration file:
```bash
sudo nano /etc/pushpin/pushpin.conf
```

2. Configure your backend routes:
```bash
sudo nano /etc/pushpin/routes
```

Example routes file:
```
# Route all requests to a backend server
* http://localhost:8080

# Route specific paths to different backends
/api http://api-server:3000
/websocket ws://websocket-server:8081
```

### Service Management

```bash
# Start Pushpin
sudo systemctl start pushpin

# Enable on boot
sudo systemctl enable pushpin

# Check status
sudo systemctl status pushpin

# View logs
sudo journalctl -u pushpin -f

# Restart service (configuration changes)
sudo systemctl restart pushpin

# Attempt reload (may not be supported)
sudo systemctl reload pushpin
```

## Configuration Files

- `/etc/pushpin/pushpin.conf` - Main configuration file
- `/etc/pushpin/routes` - Backend routing rules
- `/etc/pushpin/runner/` - Runner configuration
- `/var/lib/pushpin/` - Runtime data directory
- `/var/log/pushpin/` - Log files

## Default Ports

- **7999** - HTTP/WebSocket client connections
- **5560** - ZMQ PULL for receiving messages
- **5561** - HTTP port for receiving messages
- **5562** - ZMQ SUB for receiving messages  
- **5563** - ZMQ REP for receiving commands

## Features

This RPM package includes:
- All Pushpin binaries (pushpin, pushpin-proxy, pushpin-handler, etc.)
- Systemd service integration with automatic restart on failure
- Security hardening through systemd directives
- Dedicated `pushpin` user for privilege separation
- Automatic release number incrementation for rebuilds

## Reload Limitations

⚠️ **Note**: Pushpin does not currently support configuration reload via SIGHUP. The systemd service attempts to send USR1 signal for reload, but this may not work. To apply configuration changes, use:

```bash
sudo systemctl restart pushpin
```

## Building from Source

This repository uses GitHub releases (not git commits) for builds. The COPR Makefile:
1. Fetches the latest release from GitHub API
2. Downloads the official release tarball
3. Builds with Rust and Qt toolchains
4. Packages into RPM with systemd integration

### Build Requirements

- rust & cargo
- gcc-c++
- qt5-qtbase-devel
- zeromq-devel
- openssl-devel
- boost-devel

## Troubleshooting

### Service won't start
```bash
# Check for port conflicts
sudo ss -tlnp | grep -E '7999|556[0-3]'

# Verify configuration
pushpin --config=/etc/pushpin/pushpin.conf --check

# Check permissions
ls -la /var/lib/pushpin /var/log/pushpin
```

### Connection issues
```bash
# Test backend connectivity
curl -v http://localhost:7999/

# Monitor logs
sudo journalctl -u pushpin -f
```

## Documentation

- [Official Pushpin Documentation](https://pushpin.org/docs/)
- [Configuration Reference](https://pushpin.org/docs/configuration/)
- [API Documentation](https://pushpin.org/docs/api/)
- [GitHub Repository](https://github.com/fastly/pushpin)

## COPR Build Status

Builds are triggered automatically when:
- New releases are published on GitHub
- Manual rebuild is triggered in COPR

The build system automatically increments release numbers for same-version rebuilds.

## License

Pushpin is licensed under the Apache-2.0 License. See the upstream [LICENSE](https://github.com/fastly/pushpin/blob/main/LICENSE) file.

## Contributing

To contribute to the packaging:
1. Fork this repository
2. Make your changes
3. Test locally with `make srpm` in `.copr/`
4. Submit a pull request

For Pushpin itself, contribute upstream at https://github.com/fastly/pushpin