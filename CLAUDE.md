# iGotify ProxmoxVE Helper Script PR

## Goal
Add iGotify as a new script to https://github.com/community-scripts/ProxmoxVE

## What is iGotify
- Companion service to Gotify that forwards notifications to iOS devices via Apple APNs
- GitHub: https://github.com/androidseb25/iGotify-Notification-Assistent
- Written in C#/.NET 10, runs on ASP.NET Core runtime
- Listens on port 8681 by default
- Pre-built zips available in GitHub releases (no need to build from source)
- Release asset naming: `iGotify-Notification-Service-{arch}-v{version}.zip` (amd64, arm64, arm)
- The main DLL is called `iGotify Notification Assist.dll` (note the spaces)

## Maintainer guidance (from MickLesk, ProxmoxVE maintainer)
- This should be a NEW script (not an addon to existing gotify script)
- Use `fetch_and_deploy_gh_release` with the prebuild zip
- Model after `install/mail-archiver-install.sh` — similar .NET app pattern
- Only needs ASP.NET Core runtime (not the full SDK) since we're using pre-built binaries
- The existing `install.sh` from the iGotify repo (https://raw.githubusercontent.com/androidseb25/iGotify-Notification-Assistent/main/install.sh) shows the native install approach for reference

## Files needed for the PR
1. `install/igotify-install.sh` — Installation script (runs inside LXC)
2. `ct/igotify.sh` — Container creation script (boilerplate, sets LXC defaults)
3. `json/igotify.json` — Website metadata for helper-scripts.com

## Reference: existing gotify install script
```bash
#!/usr/bin/env bash
# Source: https://gotify.net/
source /dev/stdin <<<"$FUNCTIONS_FILE_PATH"
color
verb_ip6
catch_errors
setting_up_container
network_check
update_os

fetch_and_deploy_gh_release "gotify" "gotify/server" "prebuild" "latest" "/opt/gotify" "gotify-linux-amd64.zip"
chmod +x /opt/gotify/gotify-linux-amd64

msg_info "Creating Service"
cat <<EOF >/etc/systemd/system/gotify.service
[Unit]
Description=Gotify
Requires=network.target
After=network.target
[Service]
Type=simple
User=root
WorkingDirectory=/opt/gotify
ExecStart=/opt/gotify/./gotify-linux-amd64
Restart=always
RestartSec=3
[Install]
WantedBy=multi-user.target
EOF
systemctl enable -q --now gotify
msg_ok "Created Service"

motd_ssh
customize
cleanup_lxc
```

## Reference: mail-archiver install script (dotnet app pattern)
```bash
#!/usr/bin/env bash
# Source: https://github.com/s1t5/mail-archiver
source /dev/stdin <<<"$FUNCTIONS_FILE_PATH"
color
verb_ip6
catch_errors
setting_up_container
network_check
update_os

msg_info "Installing Dependencies"
setup_deb822_repo \
  "microsoft" \
  "https://packages.microsoft.com/keys/microsoft-2025.asc" \
  "https://packages.microsoft.com/debian/13/prod/" \
  "trixie" \
  "main"
$STD apt install -y \
  dotnet-sdk-10.0 \
  aspnetcore-runtime-8.0
msg_ok "Installed Dependencies"

PG_VERSION="17" setup_postgresql
PG_DB_NAME="mailarchiver_db" PG_DB_USER="mailarchiver" setup_postgresql_db
fetch_and_deploy_gh_release "mail-archiver" "s1t5/mail-archiver" "tarball"

msg_info "Setting up Mail-Archiver"
mv /opt/mail-archiver /opt/mail-archiver-build
cd /opt/mail-archiver-build
$STD dotnet restore
$STD dotnet publish -c Release -o /opt/mail-archiver
cp /opt/mail-archiver-build/appsettings.json /opt/mail-archiver/appsettings.json
sed -i "s|\"DefaultConnection\": \"[^\"]*\"|\"DefaultConnection\": \"Host=localhost;Database=mailarchiver_db;Username=mailarchiver;Password=$PG_DB_PASS\"|" /opt/mail-archiver/appsettings.json
rm -rf /opt/mail-archiver-build

cat <<EOF >/opt/mail-archiver/.env
ASPNETCORE_URLS=http://+:5000
ASPNETCORE_ENVIRONMENT=Production
TZ=UTC
EOF
msg_ok "Setup Mail-Archiver"

msg_info "Creating Service"
cat <<EOF >/etc/systemd/system/mail-archiver.service
[Unit]
Description=Mail-Archiver Service
After=network.target
[Service]
EnvironmentFile=/opt/mail-archiver/.env
WorkingDirectory=/opt/mail-archiver
ExecStart=/usr/bin/dotnet MailArchiver.dll
Restart=always
[Install]
WantedBy=multi-user.target
EOF
systemctl enable -q --now mail-archiver
msg_info "Created Service"

motd_ssh
customize
cleanup_lxc
```

## Key differences from mail-archiver
- iGotify has PRE-BUILT binaries (no dotnet restore/publish needed)
- Only needs `aspnetcore-runtime-10.0` (not the full SDK)
- No database (no PostgreSQL)
- Port 8681 instead of 5000

## Things to verify
- Exact `fetch_and_deploy_gh_release` arguments for the prebuild zip filename matching
- Look at other ct/*.sh files and json/*.json files in the repo to match the exact format
- Check if the repo uses `trixie` or `bookworm` as the default Debian version for new scripts
