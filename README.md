**Unified Technical Specification: IPTV Catch-Up Streaming System (Local Proxy + Rails Backend)**

---

## 1. Overview

This document defines the full-stack technical requirements for an IPTV Catch-Up system, combining Ruby on Rails–based backend services with a Debian 12–based local proxy server deployed in hotels. The goal is to provide high-performance, bandwidth-efficient HLS catch-up functionality with features like EPG navigation, trick play (seek/rewind/fast‑forward), advertisement detection, accessibility support, and future extensibility to cloud‑based adaptive bitrate streaming.

---

## 1.1 Prerequisites

The following packages and configurations are required on a clean Debian 12 system before installing the IPTV Catch-Up stack:

### Required Packages

```bash
apt update && apt install -y \
  sudo curl wget git ffmpeg gnupg2 \
  build-essential libpq-dev ruby-full \
  redis-server postgresql nginx libvips \
  cron unzip net-tools yq
```

### User Setup

* Create a system user for the streaming app and grant passwordless sudo:

  ```bash
  adduser --disabled-password --gecos "Catchup Proxy" catchup
  usermod -aG sudo catchup
  echo 'catchup ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/catchup
  chmod 440 /etc/sudoers.d/catchup
  ```

### Directories

* Ensure these paths are writable:

  * `/var/www/catchup`
  * `/var/log/catchup`
  * `/var/epg_incoming`
  * `/var/tmp/ffmpeg`

### Time Sync

* Enable NTP for accurate EPG alignment:

  ```bash
  apt install -y systemd-timesyncd
  timedatectl set-ntp true
  ```

### Ruby on Rails Installation (Debian 12)

Install Ruby (3.2+) and Rails with PostgreSQL support:

```bash
su - catchup
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/main/bin/rbenv-installer | bash
printf '\nexport PATH="$HOME/.rbenv/bin:$PATH"\neval "$(rbenv init - bash)"\n' >> ~/.bashrc
source ~/.bashrc
rbenv install 3.2.2
rbenv global 3.2.2
gem install bundler
bundle config set --local without 'development test'
gem install rails -v 7.1.3
rbenv rehash
```

Install Node.js and Yarn:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs
npm install -g yarn
```

Install PostgreSQL headers:

```bash
apt install -y libpq-dev
```

### Optional (Dev/Test)

```bash
apt install -y nodejs
```

### Swap (if <8 GB RAM)

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
dd if=/dev/zero of=/swapfile bs=1M count=2048
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

### Log Rotation

Create `/etc/logrotate.d/catchup`:

```text
/var/log/catchup/*.log {
  daily
  rotate 7
  compress
  missingok
  notifempty
  copytruncate
}
```

### Journald Tuning (Optional)

```bash
mkdir -p /var/log/journal
sed -i 's/#Storage=.*/Storage=persistent/' /etc/systemd/journald.conf
sed -i 's/#SystemMaxUse=.*/SystemMaxUse=200M/' /etc/systemd/journald.conf
systemctl restart systemd-journald
```

---

## 2. System Architecture

### 2.1 Rails Backend

* **Framework**: Ruby on Rails 7.x
* **Web Server**: Puma + Nginx (reverse proxy)
* **WebSocket**: ActionCable + Redis
* **Background Jobs**: Sidekiq
* **Streaming**: FFmpeg for HLS segment generation
* **Database**: PostgreSQL (metadata), Redis (cache)
* **Storage**: Local FS (`/public/streams`) or optional S3

### 2.2 Local Proxy (Hotel Side)

* **Input**: HTTP unicast (default) or UDP multicast
* **OS**: Debian 12
* **Segments**: FFmpeg → `.ts` + `.m3u8`
* **Serve**: Nginx or embedded Ruby server
* **Dir Structure**: `/catchup/<channel_id>/<YYYY-MM-DD>/*.ts`

### 2.3 Architecture Diagram

```plaintext
+-------------+        +-------------------+        +------------------------+
| Google TV   | <----> | Hotel Local Proxy | <----> | HTTP TV Stream Sources |
| (Browser)   |        | (Debian 12 + Nginx)|        | + XML EPG             |
+-------------+        +-------------------+        +------------------------+
       ^                       ^                             ^
       |                       |                             |
       v                       v                             v
 Rails Backend <------------ Redis/WebSocket         Cloud ABR (future)
(Puma, Sidekiq, PostgreSQL)

```

### 2.4 FFmpeg & Sidekiq Service Management (systemd)

To ensure reliability and automatic recovery, each catch-up stream and job processor runs under `systemd` supervision.

#### FFmpeg Unit (`catchup-ffmpeg-<CHANNEL>.service`)

```ini
[Unit]
Description=Catch-Up FFmpeg Segmenter for Channel <CHANNEL>
After=network.target

[Service]
User=catchup
ExecStart=/usr/local/bin/catchup_ffmpeg_wrapper.sh <CHANNEL> <INPUT_URL>
Restart=on-failure
RestartSec=5
StandardOutput=append:/var/log/catchup/<CHANNEL>.log
StandardError=inherit

[Install]
WantedBy=multi-user.target
```

#### Wrapper Script: `/usr/local/bin/catchup_ffmpeg_wrapper.sh`

```bash
#!/bin/bash
CHANNEL_ID="$1"
INPUT_URL="$2"
DATE=$(date +%F)
OUTDIR="/catchup/$CHANNEL_ID/$DATE"
mkdir -p "$OUTDIR"
exec ffmpeg -re -i "$INPUT_URL" \
  -c:v copy -c:a copy -f hls \
  -hls_time 10 -hls_list_size 5 -hls_segment_type mpegts \
  -hls_flags delete_segments \
  "$OUTDIR/playlist.m3u8"
```

#### Sidekiq Service (`sidekiq.service`)

```ini
[Unit]
Description=Sidekiq Background Jobs for IPTV Catchup
After=network.target redis.service postgresql.service

[Service]
User=catchup
WorkingDirectory=/var/www/catchup
ExecStart=/home/catchup/.rbenv/shims/bundle exec sidekiq -e production -C config/sidekiq.yml
Environment=RAILS_ENV=production
Restart=on-failure
KillSignal=SIGTERM
TimeoutStopSec=15

[Install]
WantedBy=multi-user.target
```

#### Aggregate Target (`catchup-ffmpeg.target`)

```ini
[Unit]
Description=All Catch-Up FFmpeg Services
Requires=catchup-ffmpeg-*.service
```

---

### 2.5 Bash Unit Generator from YAML Config

Script: `generate_ffmpeg_units.sh`

```bash
#!/bin/bash
CONFIG_FILE="$1"
SYSTEMD_DIR="/etc/systemd/system"
WRAPPER_SCRIPT="/usr/local/bin/catchup_ffmpeg_wrapper.sh"

[[ -z "$CONFIG_FILE" || ! -f "$CONFIG_FILE" ]] && { echo "Usage: $0 channels.yml"; exit 1; }
command -v yq >/dev/null || { echo "Install yq: apt install yq"; exit 1; }

UNITS=()
for CH in $(yq '.channels|keys|.[]' "$CONFIG_FILE"); do
  URL=$(yq ".channels.\"$CH\".input" "$CONFIG_FILE")
  UFILE="$SYSTEMD_DIR/catchup-ffmpeg-$CH.service"
  LOG="/var/log/catchup/$CH.log"

  cat > "$UFILE" <<EOF
[Unit]
Description=FFmpeg Segmenter for $CH
After=network.target

[Service]
User=catchup
ExecStart=$WRAPPER_SCRIPT $CH $URL
Restart=on-failure
RestartSec=5
StandardOutput=append:$LOG
EOF
  chmod 644 "$UFILE"
  UNITS+=("catchup-ffmpeg-$CH.service")
done

# Generate target
TFILE="$SYSTEMD_DIR/catchup-ffmpeg.target"
echo -e "[Unit]\nDescription=All Catch-Up FFmpeg Services" > $TFILE
for u in "${UNITS[@]}"; do echo "Requires=$u" >> $TFILE; done
systemctl daemon-reload
echo "Units generated and reloaded."
```

---

## 3. Core Functional Components

### 3.1 Channel & Stream Management

* Only channels listed in YAML are recorded for catch-up.
* EPG metadata ingested for all channels; only configured inputs are segmented.
* Toggle catch-up per channel; live+catch-up coexist.

```bash
ffmpeg -i http://source/stream \
  -c:v h264 -c:a aac -f hls \
  -hls_time 10 -hls_list_size 5 \
  /catchup/CH1/$(date +%F)/playlist.m3u8
```

### 3.2 Background Task Management

* **EPG Import**: Sidekiq job runs every 20–30 min or on file change
* **CleanupOldSegmentsJob**: Purges `.ts`/`.m3u8` older than retention
* **Sidekiq**: Managed by `systemd`, auto-restarts on failure
* **Manual Cleanup**: `RAILS_ENV=production bundle exec rake catchup:cleanup`

### 3.3 HLS Storage

#### Configuration (YAML)

```yaml
channels:
  CH34:
    input: "http://..."
    retention_days: 7
  CH88:
    input: "udp://..."
    retention_days: 10
default:
  retention_days: 7
hls_segment_duration: 10
storage_quota_gb: 1000
catchup_directory: /catchup
```

#### Sidekiq Cleanup Job Example

```ruby
class CleanupOldSegmentsJob < ApplicationJob
  queue_as :low
  def perform
    cutoff = Time.now - Rails.configuration.x.catchup.retention_days.days
    Dir.glob("#{Rails.root}/public/catchup/**/*.ts").each { |f| File.delete(f) if File.mtime(f) < cutoff }
    Dir.glob("#{Rails.root}/public/catchup/**/*.m3u8").each { |f| File.delete(f) if File.mtime(f) < cutoff }
  end
end
```

#### Rake Wrapper

```ruby
task cleanup: :environment do
  CleanupOldSegmentsJob.perform_now
end
```

#### Redis Lock

```ruby
def perform
  return unless Redis.current.setnx("cleanup_lock", Time.now.to_i)
  super
ensure
  Redis.current.del("cleanup_lock")
end
```

---

## 4. HLS Storage

(Full details already covered under 3.3)

## 5. Video Segment Cleanup

(As above)

## 6. Logging and Monitoring

* Persist logs via `journald` and `logrotate`
* Monitor via `prometheus-node-exporter` or `collectd`
* Add `/healthz` endpoint if needed

## 7. Network Configuration

* Nginx config for secure local delivery
* Disable `autoindex`
* Proxy cache tuning

## 8. Admin Interface (Future)

* Rails Admin + Swagger UI
* Protected by basic auth

## 9. Access Control (Optional)

* API tokens
* LAN IP allowlist

## 10. Puppet Deployment (Optional)

### Sample Puppet Manifest

```puppet
class catchup::install {
  package { ['ffmpeg','ruby-full','libpq-dev']: ensure => installed }
  vcsrepo { '/var/www/catchup': source => 'https://git...'}
  exec { 'bundle install': cwd => '/var/www/catchup', command => 'bundle install' }
  service { ['streaming-web','sidekiq','redis']: ensure=>running, enable=>true }
}
```

## 11. CI/CD Pipeline (GitHub Actions)

```yaml
name: CI/CD
on: push
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: ruby/setup-ruby@v1
        with: {ruby-version: '3.2'}
      - run: bundle install
      - run: bundle exec rspec
      - uses: appleboy/ssh-action@master
        with:
          host: ${{secrets.SSH_HOST}}
          key: ${{secrets.SSH_KEY}}
          script: |
            cd /var/www/catchup
            git pull
            bundle install
            RAILS_ENV=production bundle exec rake db:migrate
            systemctl restart streaming-web sidekiq
```

## 12. EPG Processing Logic

* Watch `/var/epg_incoming/epg.xml`
* Parse `<programme>` via Nokogiri
* Upsert into `epgs` table

## 13. Recommendations for Scalable Deployment

* Use `-c copy` when possible
* 8 s segment duration
* SSD storage (≥2 TB)
* RAID or cold spares

## 14. API for Android TV / Google TV

* `GET /api/v1/epg?channel_id=&date=`
* `GET /api/v1/stream/start?channel_id=&timestamp=`
* `GET /api/v1/stream/live?channel_id=`
* `POST /api/v1/analytics/playback`

## 15. OpenAPI Spec & Swagger UI

(Embed `swagger.yaml` + UI under `/docs`)

## 16. Final Notes

This specification merges Rails backend and Debian proxy requirements into a single, modular IPTV Catch‑Up system. Future artifacts (wireframes, multi‑site dashboard) may follow.

---

## Appendix A: Phase 1 Setup Script

Use this script to bootstrap a Debian 12 server for the IPTV catch‑up stack (Phase 1). Save as `phase1_setup.sh`, `chmod +x`, then run `sudo ./phase1_setup.sh`.

```bash
#!/usr/bin/env bash
# Phase 1: System Setup & Prerequisites
# Usage: sudo ./phase1_setup.sh

set -euo pipefail

if (( EUID != 0 )); then
  echo "Please run as root: sudo $0"
  exit 1
fi

echo "=== Phase 1: System Setup & Prerequisites ==="

# Update & upgrade
apt update && apt upgrade -y

# Install adduser & sudo
apt install -y adduser sudo

# Create catchup user
if ! id catchup &>/dev/null; then
  adduser --disabled-password --gecos "Catchup Proxy" catchup
  usermod -aG sudo catchup
  echo 'catchup ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/catchup
  chmod 440 /etc/sudoers.d/catchup
fi

# Core dependencies
apt install -y curl wget git ffmpeg gnupg2 \
  build-essential libpq-dev ruby-full \
  redis-server postgresql nginx libvips \
  cron unzip net-tools yq openssh-server

# Enable NTP
apt install -y systemd-timesyncd
timedatectl set-ntp true

# Swap if <8 GB RAM
MIN_KB=8388608; RAM_KB=$(awk '/MemTotal/ {print $2}' /proc/meminfo)
if (( RAM_KB < MIN_KB )); then
  fallocate -l 2G /swapfile
  chmod 600 /swapfile
  mkswap /swapfile
  swapon /swapfile
  echo '/swapfile none swap sw 0 0' >> /etc/fstab
fi

# App directories
for d in /var/www/catchup /var/log/catchup /var/epg_incoming /var/tmp/ffmpeg; do
  mkdir -p "$d"
  chown catchup:catchup "$d"
done

# Logrotate
cat > /etc/logrotate.d/catchup <<'EOF'
/var/log/catchup/*.log {
  daily
  rotate 7
  compress
  missingok
  notifempty
  copytruncate
}
EOF

# Persistent journald
mkdir -p /var/log/journal
sed -i 's/^#Storage=.*/Storage=persistent/' /etc/systemd/journald.conf
sed -i 's/^#SystemMaxUse=.*/SystemMaxUse=200M/' /etc/systemd/journald.conf
systemctl restart systemd-journald

# Verify services
systemctl is-active postgresql && echo "PostgreSQL running"
systemctl is-active redis-server && echo "Redis running"
systemctl is-enabled nginx && echo "Nginx enabled"

echo "=== Phase 1 complete ==="
```

## Appendix B: Phase 2 Setup Script

Use this script to deploy the Rails backend (Phase 2). Provide your repo URL: `sudo ./phase2_setup.sh <git_repo_url>`. Save as `phase2_setup.sh`, `chmod +x`, then run.

```bash
#!/usr/bin/env bash
# Phase 2: Rails Backend Setup
# Usage: sudo ./phase2_setup.sh https://github.com/your-org/CatchUP.git

set -euo pipefail

if (( EUID != 0 )); then
  echo "Please run as root: sudo $0 <git_repo_url>"
  exit 1
fi
if [[ $# -lt 1 ]]; then
  echo "Usage: sudo $0 https://github.com/your-org/CatchUP.git"
  exit 1
fi

REPO_URL="$1"
TMP_DIR=$(mktemp -d)
DEST="/var/www/catchup"
RAILS_ENV=${RAILS_ENV:-production}
RBENV_ROOT="/home/catchup/.rbenv"
RUBY_VER="3.2.2"
DB_NAME="catchup_${RAILS_ENV}"

echo "=== Phase 2: Rails Backend Setup ==="
echo "Repo: $REPO_URL"

# Clone
git clone --depth 1 "$REPO_URL" "$TMP_DIR"

# Find Gemfile
GEMDIR=$(find "$TMP_DIR" -type f -name Gemfile -exec dirname {} \; | head -n1 || true)
if [[ -z "$GEMDIR" ]]; then
  echo "Gemfile not found in repo"
  rm -rf "$TMP_DIR"
  exit 1
fi
echo "Found app in: ${GEMDIR#$TMP_DIR/}"

# Copy to dest
rm -rf "$DEST"
mkdir -p "$DEST"
cp -R "$GEMDIR/." "$DEST"
chown -R catchup:catchup "$DEST"
rm -rf "$TMP_DIR"

# System deps
apt update
apt install -y git curl build-essential \
  libssl-dev libreadline-dev zlib1g-dev \
  libsqlite3-dev libpq-dev libffi-dev libyaml-dev \
  nodejs npm yarn openssh-client

# Postgres & Redis
systemctl enable --now postgresql redis-server

# Create DB/role if needed
sudo -u postgres psql -tAc "SELECT 1 FROM pg_roles WHERE rolname='catchup'" | grep -q 1 || sudo -u postgres psql -c "CREATE ROLE catchup LOGIN PASSWORD 'catchup';"
sudo -u postgres psql -tAc "SELECT 1 FROM pg_database WHERE datname='$DB_NAME'" | grep -q 1 || sudo -u postgres psql -c "CREATE DATABASE $DB_NAME OWNER catchup;"

# rbenv
if [ ! -d "$RBENV_ROOT" ]; then
  runuser -l catchup -c 'git clone https://github.com/rbenv/rbenv.git ~/.rbenv'
fi
if [ ! -d "$RBENV_ROOT/plugins/ruby-build" ]; then
  runuser -l catchup -c 'git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build'
fi
grep -qxF 'export PATH="$HOME/.rbenv/bin:$PATH"' /home/catchup/.bashrc || echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> /home/catchup/.bashrc
grep -qxF 'eval "$(rbenv init - bash)"' /home/catchup/.bashrc || echo 'eval "$(rbenv init - bash)"' >> /home/catchup/.bashrc

# Install Ruby
runuser -l catchup -c "export PATH=\"$RBENV_ROOT/bin:\$PATH\" && eval \"\$(rbenv init - bash)\" && rbenv install -s $RUBY_VER && rbenv global $RUBY_VER"

# Gems & Rails
runuser -l catchup -c "export PATH=\"$RBENV_ROOT/shims:$RBENV_ROOT/bin:\$PATH\" && gem install bundler --no-document && gem install rails -v 7.1.3 --no-document && rbenv rehash"

# Bundle & yarn
runuser -l catchup -c "cd $DEST && export PATH=\"$RBENV_ROOT/shims:$RBENV_ROOT/bin:\$PATH\" && bundle config set --local without 'development test' && bundle install && yarn install --check-files"

# DB config
echo -e "$RAILS_ENV:
  adapter: postgresql
  encoding: unicode
  database: $DB_NAME
  pool: 5
  username: catchup
  password: catchup
  host: localhost" > $DEST/config/database.yml
chown catchup:catchup $DEST/config/database.yml
chmod 600 $DEST/config/database.yml

# Setup & assets
runuser -l catchup -c "cd $DEST && export PATH=\"$RBENV_ROOT/shims:$RBENV_ROOT/bin:\$PATH\" && rails db:setup && rails assets:precompile"

# Puma service
cat > /etc/systemd/system/streaming-web.service <<EOF
[Unit]
Description=Puma HTTP Server for Catchup
After=network.target

[Service]
Type=simple
User=catchup
WorkingDirectory=$DEST
ExecStart=$RBENV_ROOT/shims/bundle exec puma -C config/puma.rb
Environment=RAILS_ENV=$RAILS_ENV
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# Enable
systemctl daemon-reload
systemctl enable --now streaming-web.service sidekiq.service

echo "=== Phase 2 complete ==="
```

