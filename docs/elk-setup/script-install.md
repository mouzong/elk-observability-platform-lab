## ELK Installation Script on Oracle Linux 8.10

### 1 - Prepare Oracle Linux

First start by verifying the OS 
```bash
# Check the release of the OS 
cat /etc/oracle-release

# get the CPUI architecture
uname -m
```

The expected OS is Oracle Linux Server release 8.10. The commands below assume x86_64. For an ARM64 guest, use the corresponding aarch64 Elastic archives.

Install the required utilities:

```bash
sudo dnf update -y

sudo dnf install -y \
  curl \
  wget \
  tar \
  gzip \
  unzip \
  vim \
  jq \
  procps-ng \
  firewalld
```

Enable the firewall:

```bash
sudo systemctl enable --now firewalld
```

Configure kernel parameters
Elasticsearch uses memory-mapped files and needs an appropriate vm.max_map_count.

```bash
sudo tee /etc/sysctl.d/99-elasticsearch.conf > /dev/null <<'EOF'
vm.max_map_count=1048576
EOF

sudo sysctl --system
sysctl vm.max_map_count
```

The resulting value should be `1048576`.

### 2 - Create the dedicated elk account

Create a system account without interactive login privileges:

```bash
sudo groupadd --system elk

sudo useradd \
  --system \
  --gid elk \
  --home-dir /opt/elk \
  --shell /sbin/nologin \
  elk
```

Create the directory structure:

```bash
sudo mkdir -p \
  /opt/elk \
  /etc/elk/{elasticsearch,logstash,kibana} \
  /var/lib/elk/{elasticsearch,logstash,kibana} \
  /var/log/elk/{elasticsearch,logstash,kibana}
```

Apply ownership:

```bash
sudo chown -R elk:elk \
  /opt/elk \
  /var/lib/elk \
  /var/log/elk
```

Set basic permissions:

```bash
sudo chmod 750 /opt/elk
sudo chmod -R 750 /var/lib/elk /var/log/elk
sudo chown root:elk /etc/elk
sudo chmod 750 /etc/elk
sudo chown -R root:elk /etc/elk/*
sudo chmod -R 750 /etc/elk/*
```

We will keep configuration files owned by `root`, readable by the `elk` group, and not writable by the running services.

### 3 - Download and install the ELK binaries

Set the versions:

```bash
export ELK_VERSION="8.19.23"
cd /tmp
```

Download the archives:

```bash
# For amd64 architecture
for component in elasticsearch logstash kibana filebeat metricbeat; do
  curl -fLO \
    "https://artifacts.elastic.co/downloads/${component}/${component}-${ELK_VERSION}-linux-x86_64.tar.gz"
done

# For arm64 architecture use the links below
for component in elasticsearch logstash kibana filebeat metricbeat; do
  curl -fLO \
    "https://artifacts.elastic.co/downloads/${component}/${component}-${ELK_VERSION}-linux-aarch64.tar.gz"
done
```

Extract them:

```bash
sudo tar -xzf elasticsearch-${ELK_VERSION}-linux-x86_64.tar.gz \
  -C /opt/elk

sudo tar -xzf logstash-${ELK_VERSION}-linux-x86_64.tar.gz \
  -C /opt/elk

sudo tar -xzf kibana-${ELK_VERSION}-linux-x86_64.tar.gz \
  -C /opt/elk
```

Create stable symlinks:

```bash
sudo ln -s /opt/elk/elasticsearch-${ELK_VERSION} \
  /opt/elk/elasticsearch

sudo ln -s /opt/elk/logstash-${ELK_VERSION} \
  /opt/elk/logstash

sudo ln -s /opt/elk/kibana-${ELK_VERSION} \
  /opt/elk/kibana
```

Correct the archive ownership:

```bash
sudo chown -R elk:elk \
  /opt/elk/elasticsearch* \
  /opt/elk/logstash* \
  /opt/elk/kibana*
```

### 4 - Configure Elasticsearch

For the initial deployment, we'll build a secured single-node cluster. You can extend it to three Elasticsearch VMs afterward.

### 4.1 - Prepare the configuration directory

Copy the Elasticsearch configuration, including its JVM and logging settings:

```bash
sudo cp -a /opt/elk/elasticsearch/config/. \
  /etc/elk/elasticsearch/

sudo chown -R elk:elk /etc/elk/elasticsearch
sudo chmod 750 /etc/elk/elasticsearch
```

Elasticsearch needs temporary write access to its configuration directory for first-start security auto-configuration. After enrollment, we'll tighten those permissions.

Create the configuration:

```bash
sudo tee /etc/elk/elasticsearch/elasticsearch.yml > /dev/null <<'EOF'
cluster.name: codegrill-observability
node.name: es01

path.data: /var/lib/elk/elasticsearch
path.logs: /var/log/elk/elasticsearch

network.host: 127.0.0.1
http.port: 9200

discovery.type: single-node
EOF

sudo chown elk:elk /etc/elk/elasticsearch/elasticsearch.yml
```

network.host: `127.0.0.1` is suitable when all components run on the same VM. For your multi-VM architecture, you'll later replace this with the appropriate private interface and configure inter-node TLS and discovery.

### 4.2 - Configure JVM heap

For an Elasticsearch VM with `8GB RAM`, begin with a `4GB heap`:

```bash
sudo mkdir -p /etc/elk/elasticsearch/jvm.options.d

sudo tee /etc/elk/elasticsearch/jvm.options.d/heap.options > /dev/null <<'EOF'
-Xms4g
-Xmx4g
EOF
```

Remove any conflicting heap overrides and ensure the VM has sufficient memory for the OS and filesystem cache.

### 5.3 - Create the Elasticsearch systemd unit

Create `/etc/systemd/system/elasticsearch.service`:

```ini
[Unit]
Description=Elasticsearch 8.19.23
Documentation=https://www.elastic.co/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple

User=elk
Group=elk

WorkingDirectory=/opt/elk/elasticsearch

Environment="ES_HOME=/opt/elk/elasticsearch"
Environment="ES_PATH_CONF=/etc/elk/elasticsearch"

ExecStart=/opt/elk/elasticsearch/bin/elasticsearch

Restart=on-failure
RestartSec=10

LimitNOFILE=65535
LimitNPROC=4096
LimitMEMLOCK=infinity

TimeoutStartSec=900
TimeoutStopSec=120

SuccessExitStatus=143
KillSignal=SIGTERM

StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```