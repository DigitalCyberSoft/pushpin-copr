# COPR Makefile for automated pushpin builds from GitHub releases
# Builds RPMs from official Pushpin releases rather than git commits

# COPR provides outdir, set default for local testing
outdir ?= .

# Default target for COPR
srpm:
	@echo "=== Starting Pushpin SRPM build from GitHub releases ==="
	@echo "Output directory: $(outdir)"
	@echo "Working directory: $$(pwd)"
	@echo "User: $$(whoami)"

	# Install required tools if not available
	@if ! command -v jq >/dev/null 2>&1; then \
		echo "=== Installing jq for JSON parsing ==="; \
		if command -v dnf >/dev/null 2>&1; then \
			dnf install -y jq || yum install -y jq || exit 1; \
		else \
			yum install -y jq || exit 1; \
		fi; \
	fi

	# Get latest release version from GitHub API
	@echo "=== Fetching latest Pushpin release from GitHub ==="; \
	RELEASE_INFO=$$(curl -s https://api.github.com/repos/fastly/pushpin/releases/latest) || exit 1; \
	VERSION=$$(echo "$$RELEASE_INFO" | jq -r '.tag_name' | sed 's/^v//') || exit 1; \
	if [ -z "$$VERSION" ] || [ "$$VERSION" = "null" ]; then \
		echo "ERROR: Failed to get latest version from GitHub"; \
		exit 1; \
	fi; \
	echo "Latest Pushpin version: $$VERSION"; \
	\
	echo "=== Downloading Pushpin $$VERSION source tarball ==="; \
	curl -L -o pushpin-$$VERSION.tar.gz \
		"https://github.com/fastly/pushpin/archive/refs/tags/v$$VERSION.tar.gz" || exit 1; \
	\
	echo "=== Determining release number ==="; \
	FEDORA_VER=$$(rpm -E %fedora 2>/dev/null || echo "42"); \
	echo "Checking COPR for existing builds of version $$VERSION on Fedora $$FEDORA_VER"; \
	COPR_USER="$${COPR_USER:-reversejames}"; \
	COPR_PROJECT="$${COPR_PROJECT:-pushpin}"; \
	COPR_URL="https://download.copr.fedorainfracloud.org/results/$$COPR_USER/$$COPR_PROJECT/fedora-$$FEDORA_VER-x86_64"; \
	BUILD_DIRS=$$(curl -s "$$COPR_URL/" 2>/dev/null | grep -oE '[0-9]+-pushpin' | head -10); \
	EXISTING_RELEASES=""; \
	for dir in $$BUILD_DIRS; do \
		RPMS=$$(curl -s "$$COPR_URL/$$dir/" 2>/dev/null | grep -oE "pushpin-$$VERSION-[0-9]+\.fc[0-9]+\.x86_64\.rpm" | head -1); \
		if [ -n "$$RPMS" ]; then \
			REL=$$(echo "$$RPMS" | sed "s/.*-\([0-9]\+\)\.fc[0-9]\+\.x86_64\.rpm/\1/"); \
			EXISTING_RELEASES="$$EXISTING_RELEASES $$REL"; \
		fi; \
	done; \
	if [ -n "$$EXISTING_RELEASES" ]; then \
		MAX_REL=$$(echo $$EXISTING_RELEASES | tr ' ' '\n' | sort -n | tail -1); \
		RELEASE_NUM=$$(($$MAX_REL + 1)); \
		echo "Found existing release $$MAX_REL, incrementing to $$RELEASE_NUM"; \
	else \
		RELEASE_NUM=1; \
		echo "No existing builds found for $$VERSION, starting with release 1"; \
	fi; \
	\
	echo "=== Creating systemd service file ==="; \
	cat > pushpin.service <<'EOF'
[Unit]
Description=Pushpin realtime reverse proxy
Documentation=https://pushpin.org/docs/
After=network.target

[Service]
Type=simple
User=pushpin
Group=pushpin
WorkingDirectory=/var/lib/pushpin
ExecStart=/usr/bin/pushpin --merge-output
ExecReload=/bin/kill -USR1 $$MAINPID
Restart=on-failure
RestartSec=5s
StandardOutput=journal
StandardError=journal
SyslogIdentifier=pushpin

# Security hardening
PrivateTmp=yes
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/pushpin /var/log/pushpin /var/run/pushpin

[Install]
WantedBy=multi-user.target
EOF
	echo "Service file created"; \
	\
	echo "=== Creating default configuration file ==="; \
	cat > pushpin.conf <<'EOF'
# Pushpin configuration file
# See https://pushpin.org/docs/configuration/ for full documentation

# Global settings
include internal.conf

# Services to start (comma-separated)
services=connmgr,proxy,handler

# HTTP port to listen on
http_port=7999

# Local mode (no external dependencies)
global_route_pool_size=1

# Default route for all requests (update to your backend)
# Example: route=* http://localhost:8080
EOF
	echo "Default configuration created"; \
	\
	echo "=== Generating RPM spec file ==="; \
	cat > pushpin.spec <<EOF
%global _hardened_build 1
%global debug_package %{nil}

Name:           pushpin
Version:        $$VERSION
Release:        $$RELEASE_NUM%{?dist}
Summary:        Reverse proxy for realtime web services

License:        Apache-2.0
URL:            https://pushpin.org/
Source0:        pushpin-%{version}.tar.gz
Source1:        pushpin.service
Source2:        pushpin.conf

BuildRequires:  gcc
BuildRequires:  gcc-c++
BuildRequires:  make
BuildRequires:  cargo
BuildRequires:  rust
BuildRequires:  openssl-devel
BuildRequires:  zeromq-devel
BuildRequires:  pcre2-devel
BuildRequires:  qt5-qtbase-devel
BuildRequires:  boost-devel
BuildRequires:  systemd-rpm-macros

Requires:       openssl
Requires:       zeromq
Requires:       pcre2
Requires(pre):  shadow-utils
Requires(post): systemd
Requires(preun): systemd
Requires(postun): systemd

%description
Pushpin is a reverse proxy server written in Rust/C++ that makes it easy to
implement WebSocket, HTTP streaming, and HTTP long-polling services. The
project is unique among realtime push solutions in that it is designed to
be transparent to clients and integrates easily into an API stack.

%prep
%setup -q -n pushpin-%{version}

%build
# Set build flags
export CARGO_HOME=.cargo
export PREFIX=%{_prefix}
export BINDIR=%{_bindir}
export LIBDIR=%{_libdir}/pushpin
export CONFIGDIR=%{_sysconfdir}/pushpin
export RUST_BACKTRACE=1

# Build with release optimizations
make RELEASE=1

%install
# Install binaries
install -D -m 755 target/release/pushpin %{buildroot}%{_bindir}/pushpin
install -D -m 755 target/release/pushpin-proxy %{buildroot}%{_bindir}/pushpin-proxy
install -D -m 755 target/release/pushpin-handler %{buildroot}%{_bindir}/pushpin-handler
install -D -m 755 target/release/pushpin-connmgr %{buildroot}%{_bindir}/pushpin-connmgr
install -D -m 755 target/release/m2adapter %{buildroot}%{_bindir}/m2adapter
install -D -m 755 target/release/pushpin-publish %{buildroot}%{_bindir}/pushpin-publish

# Install legacy wrapper if it exists
if [ -f target/release/pushpin-legacy ]; then
    install -D -m 755 target/release/pushpin-legacy %{buildroot}%{_bindir}/pushpin-legacy
fi

# Create symlink for condure
ln -sf pushpin-connmgr %{buildroot}%{_bindir}/pushpin-condure

# Install configuration files
install -D -m 644 %{SOURCE2} %{buildroot}%{_sysconfdir}/pushpin/pushpin.conf
install -D -m 644 src/internal.conf %{buildroot}%{_libdir}/pushpin/internal.conf

# Create default routes file
echo "* localhost:8080" > %{buildroot}%{_sysconfdir}/pushpin/routes

# Install systemd service file
install -D -m 644 %{SOURCE1} %{buildroot}%{_unitdir}/pushpin.service

# Create required directories
install -d -m 755 %{buildroot}%{_localstatedir}/lib/pushpin
install -d -m 755 %{buildroot}%{_localstatedir}/log/pushpin
install -d -m 755 %{buildroot}%{_localstatedir}/run/pushpin
install -d -m 755 %{buildroot}%{_sysconfdir}/pushpin/runner
install -d -m 755 %{buildroot}%{_sysconfdir}/pushpin/runner/certs

# Install runner templates if they exist
if [ -d src/runner ]; then
    cp -r src/runner/* %{buildroot}%{_libdir}/pushpin/runner/ 2>/dev/null || true
fi

%pre
getent group pushpin >/dev/null || groupadd -r pushpin
getent passwd pushpin >/dev/null || \
    useradd -r -g pushpin -d /var/lib/pushpin -s /sbin/nologin \
    -c "Pushpin realtime proxy" pushpin
exit 0

%post
%systemd_post pushpin.service
if [ \$1 -eq 1 ]; then
    echo "Pushpin installed. Edit /etc/pushpin/pushpin.conf and /etc/pushpin/routes,"
    echo "then start with: systemctl start pushpin"
fi

%preun
%systemd_preun pushpin.service

%postun
%systemd_postun_with_restart pushpin.service
if [ \$1 -eq 0 ]; then
    # Package removal, not upgrade
    userdel pushpin >/dev/null 2>&1 || true
    groupdel pushpin >/dev/null 2>&1 || true
fi

%files
%license LICENSE
%doc README.md CHANGELOG.md
%{_bindir}/pushpin
%{_bindir}/pushpin-proxy
%{_bindir}/pushpin-handler
%{_bindir}/pushpin-connmgr
%{_bindir}/pushpin-condure
%{_bindir}/m2adapter
%{_bindir}/pushpin-publish
%{_bindir}/pushpin-legacy
%dir %{_libdir}/pushpin
%{_libdir}/pushpin/*
%{_unitdir}/pushpin.service
%dir %{_sysconfdir}/pushpin
%config(noreplace) %{_sysconfdir}/pushpin/pushpin.conf
%config(noreplace) %{_sysconfdir}/pushpin/routes
%dir %{_sysconfdir}/pushpin/runner
%dir %{_sysconfdir}/pushpin/runner/certs
%attr(0755,pushpin,pushpin) %dir %{_localstatedir}/lib/pushpin
%attr(0755,pushpin,pushpin) %dir %{_localstatedir}/log/pushpin
%attr(0755,pushpin,pushpin) %dir %{_localstatedir}/run/pushpin

%changelog
* $$(date '+%a %b %d %Y') COPR Builder - $$VERSION-$$RELEASE_NUM
- Automated build from GitHub release v$$VERSION
- Built from upstream Pushpin project
EOF
	echo "Spec file generated"; \
	\
	echo "=== Building SRPM ==="; \
	rpmbuild -bs \
		--define "_sourcedir $$(pwd)" \
		--define "_specdir $$(pwd)" \
		--define "_builddir $$(pwd)" \
		--define "_srcrpmdir $(outdir)" \
		--define "_rpmdir $$(pwd)" \
		pushpin.spec || exit 1; \
	\
	echo "=== SRPM build complete ==="; \
	ls -la $(outdir)/*.src.rpm

clean:
	@rm -f *.tar.gz *.spec *.src.rpm *.rpm *.service *.conf
	@rm -rf pushpin-*/ .cargo/

.PHONY: srpm clean