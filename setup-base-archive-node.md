# Base Archive Node Setup Guide

This guide will help you set up a complete Base archive node on your local server with 1TB storage.

## Prerequisites

- **Storage**: 1TB available (Base currently uses ~200-500GB)
- **RAM**: 32GB minimum (64GB recommended) 
- **CPU**: Modern multi-core CPU with good single-core performance
- **Network**: Stable internet connection with good bandwidth
- **OS**: Linux (Ubuntu 20.04+ recommended)

## Overview

Base uses the OP Stack (Optimism's technology stack), so we'll be running:
1. **op-geth** - The execution client (handles transactions/state)
2. **op-node** - The consensus client (derives L2 blocks from L1)

## Step 1: Install Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y build-essential git curl wget jq

# Install Go (required for building the clients)
wget https://go.dev/dl/go1.21.5.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.21.5.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc

# Verify Go installation
go version
```

## Step 2: Create Directories

```bash
# Create base directory for the node
mkdir -p ~/base-node
cd ~/base-node

# Create data directories
mkdir -p data/op-geth
mkdir -p data/op-node
mkdir -p logs
```

## Step 3: Build op-geth (Base's Execution Client)

```bash
# Clone op-geth repository
git clone https://github.com/ethereum-optimism/op-geth.git
cd op-geth

# Build op-geth
make geth

# Verify build
./build/bin/geth version

cd ..
```

## Step 4: Build op-node (Base's Consensus Client)

```bash
# Clone optimism repository
git clone https://github.com/ethereum-optimism/optimism.git
cd optimism

# Build op-node
make op-node

# Verify build
./bin/op-node --version

cd ..
```

## Step 5: Generate JWT Secret

The clients need a shared secret for secure communication:

```bash
# Generate JWT secret
openssl rand -hex 32 > jwt-secret.txt

# Set proper permissions
chmod 600 jwt-secret.txt
```

## Step 6: Configure Base Network Settings

Create configuration files:

```bash
# Create op-geth config
cat > geth-config.txt << 'EOF'
# Base Mainnet Configuration
--datadir=./data/op-geth
--http
--http.addr=0.0.0.0
--http.port=8545
--http.api=eth,net,web3,debug,txpool
--http.corsdomain="*"
--authrpc.addr=0.0.0.0
--authrpc.port=8551
--authrpc.jwtsecret=./jwt-secret.txt
--gcmode=archive
--syncmode=full
--rollup.sequencerhttp=https://mainnet-sequencer.base.org
--op-network=base-mainnet
--verbosity=3
--maxpeers=100
--cache=8192
--port=30303
--discovery.port=30303
EOF

# Create op-node config  
cat > node-config.txt << 'EOF'
# Base op-node configuration
--network=base-mainnet
--l1=https://eth-mainnet.g.alchemy.com/v2/YOUR_ALCHEMY_KEY
--l1.rpckind=alchemy
--l1.beacon=https://eth-mainnet.g.alchemy.com/v2/YOUR_ALCHEMY_KEY
--l2=http://localhost:8551
--l2.jwt-secret=./jwt-secret.txt
--syncmode=execution-layer
--l2.enginekind=geth
--rpc.addr=0.0.0.0
--rpc.port=9545
--metrics.enabled
--metrics.addr=0.0.0.0
--metrics.port=7300
--pprof.enabled
--log.level=info
EOF
```

## Step 7: Get Ethereum L1 RPC Access

You need access to Ethereum mainnet. Get a free API key from:
- **Alchemy**: https://www.alchemy.com/ (recommended)
- **Infura**: https://infura.io/
- **QuickNode**: https://www.quicknode.com/

Replace `YOUR_ALCHEMY_KEY` in the config files with your actual API key.

## Step 8: Download Base Snapshot (Optional but Recommended)

To speed up initial sync, download a recent snapshot:

```bash
# Check latest snapshot
curl -s https://mainnet-reth-archive-snapshots.base.org/latest

# Download latest snapshot (this will be large - several hundred GB)
# Replace the URL with the actual latest snapshot URL
wget https://mainnet-reth-archive-snapshots.base.org/$(curl -s https://mainnet-reth-archive-snapshots.base.org/latest)

# Extract snapshot to data directory
tar -xzf <snapshot-filename> -C ./data/op-geth/
```

## Step 9: Create Systemd Services

Create systemd services for automatic startup:

```bash
# Create op-geth service
sudo tee /etc/systemd/system/base-geth.service > /dev/null <<EOF
[Unit]
Description=Base op-geth Archive Node
After=network.target

[Service]
Type=simple
User=$USER
WorkingDirectory=$HOME/base-node
ExecStart=$HOME/base-node/op-geth/build/bin/geth $(cat $HOME/base-node/geth-config.txt | tr '\n' ' ')
Restart=always
RestartSec=5
StandardOutput=append:$HOME/base-node/logs/geth.log
StandardError=append:$HOME/base-node/logs/geth-error.log

[Install]
WantedBy=multi-user.target
EOF

# Create op-node service
sudo tee /etc/systemd/system/base-node.service > /dev/null <<EOF
[Unit]
Description=Base op-node
After=network.target base-geth.service
Requires=base-geth.service

[Service]
Type=simple
User=$USER
WorkingDirectory=$HOME/base-node
ExecStart=$HOME/base-node/optimism/bin/op-node $(cat $HOME/base-node/node-config.txt | tr '\n' ' ')
Restart=always
RestartSec=5
StandardOutput=append:$HOME/base-node/logs/node.log
StandardError=append:$HOME/base-node/logs/node-error.log

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd and enable services
sudo systemctl daemon-reload
sudo systemctl enable base-geth base-node
```

## Step 10: Start Your Base Archive Node

```bash
# Start op-geth first
sudo systemctl start base-geth

# Wait a few seconds, then start op-node
sleep 10
sudo systemctl start base-node

# Check status
sudo systemctl status base-geth
sudo systemctl status base-node
```

## Step 11: Monitor Your Node

```bash
# Check logs
tail -f ~/base-node/logs/geth.log
tail -f ~/base-node/logs/node.log

# Check sync progress
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' \
  http://localhost:8545

# Get latest block
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://localhost:8545
```

## Expected Sync Times

- **With snapshot**: 2-6 hours to fully sync
- **Without snapshot**: 1-3 days for full archive sync
- **Storage usage**: 200-500GB currently, growing ~100GB every 6 months

## Troubleshooting

### Common Issues:

1. **Out of Memory**: Increase swap space or add more RAM
2. **Slow Sync**: Check internet connection and L1 RPC provider
3. **Disk Space**: Monitor with `df -h` and ensure enough free space

### Useful Commands:

```bash
# Stop services
sudo systemctl stop base-node base-geth

# Restart services
sudo systemctl restart base-geth
sudo systemctl restart base-node

# Check disk usage
du -sh ~/base-node/data/

# Monitor resource usage
htop
```

## Next Steps

Once your node is synced:

1. **Enable RPC access** for your applications
2. **Set up monitoring** (Prometheus + Grafana)
3. **Configure backups** of your data directory
4. **Consider running a validator** if interested in network participation

## API Usage Examples

```bash
# Get account balance at specific block
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_getBalance","params":["0x742d35Cc6634C0532925a3b8D54C3E4f5c0C0C4d", "0x100000"],"id":1}' \
  http://localhost:8545

# Get transaction receipt
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_getTransactionReceipt","params":["0x..."],"id":1}' \
  http://localhost:8545
```

Your Base archive node will now provide complete historical access to the Base network! 