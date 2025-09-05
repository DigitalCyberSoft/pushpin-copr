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
	@if ! command -v git >/dev/null 2>&1; then \
		echo "=== Git not found, installing it ==="; \
		if command -v dnf >/dev/null 2>&1; then \
			echo "Using DNF to install git..."; \
			dnf install -y git || (echo "ERROR: Failed to install git with dnf" && exit 1); \
		elif command -v yum >/dev/null 2>&1; then \
			echo "Using YUM to install git..."; \
			yum install -y git || (echo "ERROR: Failed to install git with yum" && exit 1); \
		else \
			echo "ERROR: No package manager found to install git"; \
			exit 1; \
		fi; \
	fi
	@if ! command -v jq >/dev/null 2>&1; then \
		echo "=== jq not found, installing it ==="; \
		if command -v dnf >/dev/null 2>&1; then \
			echo "Using DNF to install jq..."; \
			dnf install -y jq || (echo "ERROR: Failed to install jq with dnf" && exit 1); \
		elif command -v yum >/dev/null 2>&1; then \
			echo "Using YUM to install jq..."; \
			yum install -y jq || (echo "ERROR: Failed to install jq with yum" && exit 1); \
		else \
			echo "ERROR: No package manager found to install jq"; \
			exit 1; \
		fi; \
	fi
	@if ! command -v qmake >/dev/null 2>&1; then \
		echo "=== qmake not found, installing it ==="; \
		if command -v dnf >/dev/null 2>&1; then \
			echo "Using DNF to install qmake..."; \
			dnf install -y qmake || (echo "ERROR: Failed to install qmake with dnf" && exit 1); \
		elif command -v yum >/dev/null 2>&1; then \
			echo "Using YUM to install qmake..."; \
			yum install -y qmake || (echo "ERROR: Failed to install qmake with yum" && exit 1); \
		else \
			echo "ERROR: No package manager found to install qmake"; \
			exit 1; \
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
	( \
	echo '[Unit]'; \
	echo 'Description=Pushpin realtime reverse proxy'; \
	echo 'Documentation=https://pushpin.org/docs/'; \
	echo 'After=network.target'; \
	echo ''; \
	echo '[Service]'; \
	echo 'Type=simple'; \
	echo 'User=pushpin'; \
	echo 'Group=pushpin'; \
	echo 'WorkingDirectory=/var/lib/pushpin'; \
	echo 'ExecStart=/usr/bin/pushpin --merge-output'; \
	echo 'ExecReload=/bin/kill -USR1 $$MAINPID'; \
	echo 'Restart=on-failure'; \
	echo 'RestartSec=5s'; \
	echo 'StandardOutput=journal'; \
	echo 'StandardError=journal'; \
	echo 'SyslogIdentifier=pushpin'; \
	echo ''; \
	echo '# Security hardening'; \
	echo 'PrivateTmp=yes'; \
	echo 'NoNewPrivileges=yes'; \
	echo 'ProtectSystem=strict'; \
	echo 'ProtectHome=yes'; \
	echo 'ReadWritePaths=/var/lib/pushpin /var/log/pushpin /var/run/pushpin'; \
	echo ''; \
	echo '[Install]'; \
	echo 'WantedBy=multi-user.target'; \
	) > pushpin.service || exit 1; \
	echo "Service file created"; \
	\
	echo "=== Creating default configuration file ==="; \
	( \
	echo '# Pushpin configuration file'; \
	echo '# See https://pushpin.org/docs/configuration/ for full documentation'; \
	echo ''; \
	echo '# Global settings'; \
	echo 'include internal.conf'; \
	echo ''; \
	echo '# Services to start (comma-separated)'; \
	echo 'services=connmgr,proxy,handler'; \
	echo ''; \
	echo '# HTTP port to listen on'; \
	echo 'http_port=7999'; \
	echo ''; \
	echo '# Local mode (no external dependencies)'; \
	echo 'global_route_pool_size=1'; \
	echo ''; \
	echo '# Default route for all requests (update to your backend)'; \
	echo '# Example: route=* http://localhost:8080'; \
	) > pushpin.conf || exit 1; \
	echo "Default configuration created"; \
	\
	echo "=== Generating RPM spec file ==="; \
	( \
	echo '%global _hardened_build 1'; \
	echo '%global debug_package %{nil}'; \
	echo ''; \
	echo "Name:           pushpin"; \
	echo "Version:        $$VERSION"; \
	echo "Release:        $$RELEASE_NUM%{?dist}"; \
	echo "Summary:        Reverse proxy for realtime web services"; \
	echo ''; \
	echo "License:        Apache-2.0"; \
	echo "URL:            https://pushpin.org/"; \
	echo "Source0:        pushpin-%{version}.tar.gz"; \
	echo "Source1:        pushpin.service"; \
	echo "Source2:        pushpin.conf"; \
	echo ''; \
	echo "BuildRequires:  gcc"; \
	echo "BuildRequires:  gcc-c++"; \
	echo "BuildRequires:  make"; \
	echo "BuildRequires:  cargo"; \
	echo "BuildRequires:  rust"; \
	echo "BuildRequires:  openssl-devel"; \
	echo "BuildRequires:  zeromq-devel"; \
	echo "BuildRequires:  pcre2-devel"; \
	echo "BuildRequires:  boost-devel"; \
	echo "BuildRequires:  systemd-rpm-macros"; \
	echo ''; \
	echo "Requires:       openssl"; \
	echo "Requires:       zeromq"; \
	echo "Requires:       pcre2"; \
	echo "Requires(pre):  shadow-utils"; \
	echo "Requires(post): systemd"; \
	echo "Requires(preun): systemd"; \
	echo "Requires(postun): systemd"; \
	echo ''; \
	echo '%description'; \
	echo 'Pushpin is a reverse proxy server written in Rust/C++ that makes it easy to'; \
	echo 'implement WebSocket, HTTP streaming, and HTTP long-polling services. The'; \
	echo 'project is unique among realtime push solutions in that it is designed to'; \
	echo 'be transparent to clients and integrates easily into an API stack.'; \
	echo ''; \
	echo '%prep'; \
	echo '%setup -q -n pushpin-%{version}'; \
	echo ''; \
	echo '%build'; \
	echo '# Set build flags'; \
	echo 'export CARGO_HOME=.cargo'; \
	echo 'export PREFIX=%{_prefix}'; \
	echo 'export BINDIR=%{_bindir}'; \
	echo 'export LIBDIR=%{_libdir}/pushpin'; \
	echo 'export CONFIGDIR=%{_sysconfdir}/pushpin'; \
	echo 'export RUST_BACKTRACE=1'; \
	echo 'export CARGO_PROFILE_RELEASE_BUILD_OVERRIDE_DEBUG=true'; \
	echo ''; \
	echo '# Build Pushpin'; \
	echo '# Fetch cargo dependencies first (needed when building from tarball)'; \
	echo 'cargo fetch'; \
	echo ''; \
	echo '# Build with release flag but without offline mode'; \
	echo '# We build components separately to avoid the --offline flag'; \
	echo 'cargo build --release'; \
	echo 'make postbuild RELEASE=1'; \
	echo ''; \
	echo '%install'; \
	echo '# Install binaries'; \
	echo 'install -D -m 755 target/release/pushpin %{buildroot}%{_bindir}/pushpin'; \
	echo 'install -D -m 755 target/release/pushpin-proxy %{buildroot}%{_bindir}/pushpin-proxy'; \
	echo 'install -D -m 755 target/release/pushpin-handler %{buildroot}%{_bindir}/pushpin-handler'; \
	echo 'install -D -m 755 target/release/pushpin-connmgr %{buildroot}%{_bindir}/pushpin-connmgr'; \
	echo 'install -D -m 755 target/release/m2adapter %{buildroot}%{_bindir}/m2adapter'; \
	echo 'install -D -m 755 target/release/pushpin-publish %{buildroot}%{_bindir}/pushpin-publish'; \
	echo ''; \
	echo '# Install legacy wrapper if it exists'; \
	echo 'if [ -f target/release/pushpin-legacy ]; then'; \
	echo '    install -D -m 755 target/release/pushpin-legacy %{buildroot}%{_bindir}/pushpin-legacy'; \
	echo 'fi'; \
	echo ''; \
	echo '# Create symlink for condure'; \
	echo 'ln -sf pushpin-connmgr %{buildroot}%{_bindir}/pushpin-condure'; \
	echo ''; \
	echo '# Install configuration files'; \
	echo 'install -D -m 644 %{SOURCE2} %{buildroot}%{_sysconfdir}/pushpin/pushpin.conf'; \
	echo 'install -D -m 644 src/internal.conf %{buildroot}%{_libdir}/pushpin/internal.conf'; \
	echo ''; \
	echo '# Create default routes file'; \
	echo 'echo "* localhost:8080" > %{buildroot}%{_sysconfdir}/pushpin/routes'; \
	echo ''; \
	echo '# Install systemd service file'; \
	echo 'install -D -m 644 %{SOURCE1} %{buildroot}%{_unitdir}/pushpin.service'; \
	echo ''; \
	echo '# Create required directories'; \
	echo 'install -d -m 755 %{buildroot}%{_localstatedir}/lib/pushpin'; \
	echo 'install -d -m 755 %{buildroot}%{_localstatedir}/log/pushpin'; \
	echo 'install -d -m 755 %{buildroot}%{_localstatedir}/run/pushpin'; \
	echo 'install -d -m 755 %{buildroot}%{_sysconfdir}/pushpin/runner'; \
	echo 'install -d -m 755 %{buildroot}%{_sysconfdir}/pushpin/runner/certs'; \
	echo ''; \
	echo '# Install runner templates if they exist'; \
	echo 'if [ -d src/runner ]; then'; \
	echo '    cp -r src/runner/* %{buildroot}%{_libdir}/pushpin/runner/ 2>/dev/null || true'; \
	echo 'fi'; \
	echo ''; \
	echo '%pre'; \
	echo 'getent group pushpin >/dev/null || groupadd -r pushpin'; \
	echo 'getent passwd pushpin >/dev/null || \'; \
	echo '    useradd -r -g pushpin -d /var/lib/pushpin -s /sbin/nologin \'; \
	echo '    -c "Pushpin realtime proxy" pushpin'; \
	echo 'exit 0'; \
	echo ''; \
	echo '%post'; \
	echo '%systemd_post pushpin.service'; \
	echo 'if [ $$1 -eq 1 ]; then'; \
	echo '    echo "Pushpin installed. Edit /etc/pushpin/pushpin.conf and /etc/pushpin/routes,"'; \
	echo '    echo "then start with: systemctl start pushpin"'; \
	echo 'fi'; \
	echo ''; \
	echo '%preun'; \
	echo '%systemd_preun pushpin.service'; \
	echo ''; \
	echo '%postun'; \
	echo '%systemd_postun_with_restart pushpin.service'; \
	echo 'if [ $$1 -eq 0 ]; then'; \
	echo '    # Package removal, not upgrade'; \
	echo '    userdel pushpin >/dev/null 2>&1 || true'; \
	echo '    groupdel pushpin >/dev/null 2>&1 || true'; \
	echo 'fi'; \
	echo ''; \
	echo '%files'; \
	echo '%license LICENSE'; \
	echo '%doc README.md CHANGELOG.md'; \
	echo '%{_bindir}/pushpin'; \
	echo '%{_bindir}/pushpin-proxy'; \
	echo '%{_bindir}/pushpin-handler'; \
	echo '%{_bindir}/pushpin-connmgr'; \
	echo '%{_bindir}/pushpin-condure'; \
	echo '%{_bindir}/m2adapter'; \
	echo '%{_bindir}/pushpin-publish'; \
	echo '%{_bindir}/pushpin-legacy'; \
	echo '%dir %{_libdir}/pushpin'; \
	echo '%{_libdir}/pushpin/*'; \
	echo '%{_unitdir}/pushpin.service'; \
	echo '%dir %{_sysconfdir}/pushpin'; \
	echo '%config(noreplace) %{_sysconfdir}/pushpin/pushpin.conf'; \
	echo '%config(noreplace) %{_sysconfdir}/pushpin/routes'; \
	echo '%dir %{_sysconfdir}/pushpin/runner'; \
	echo '%dir %{_sysconfdir}/pushpin/runner/certs'; \
	echo '%attr(0755,pushpin,pushpin) %dir %{_localstatedir}/lib/pushpin'; \
	echo '%attr(0755,pushpin,pushpin) %dir %{_localstatedir}/log/pushpin'; \
	echo '%attr(0755,pushpin,pushpin) %dir %{_localstatedir}/run/pushpin'; \
	echo ''; \
	echo '%changelog'; \
	echo "* $$(date '+%a %b %d %Y') COPR Builder - $$VERSION-$$RELEASE_NUM"; \
	echo '- Automated build from GitHub release v%{version}'; \
	echo '- Built from upstream Pushpin project'; \
	) > pushpin.spec || exit 1; \
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