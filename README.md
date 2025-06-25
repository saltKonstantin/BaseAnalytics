# BaseAnalytics

A full Base L2 archive node setup for comprehensive blockchain data analysis.

## 🎯 Project Status: **OPERATIONAL** ✅

**Base Archive Node**: Fully deployed and syncing on local server
- **Services**: `base-geth` and `base-node` running successfully
- **Sync Status**: Currently processing ~29.5M historical blocks (0% → 100%)
- **RPC Endpoint**: `http://localhost:8545` (accessible from server)
- **Node Type**: Full archive node with complete historical state

## 📋 Server Details

**Hardware**: xubuntu/CasaOS server
- **CPU**: Intel N150  
- **RAM**: 15GB + 4GB swap = 19GB total
- **Storage**: 1.7TB available (`/DATA/Storage/base-node/`)
- **OS**: Ubuntu 24.04
- **Access**: SSH access configured

## 🔧 Architecture

**OP Stack Components**:
- **`op-geth`**: Execution layer (Base L2 blockchain processing)
- **`op-node`**: Consensus layer (reads from Ethereum L1, coordinates with geth)

**Data Sources**:
- **L1 Execution**: Alchemy Ethereum mainnet RPC
- **L1 Beacon**: PublicNode beacon chain API  
- **L2 Sequencer**: Base mainnet sequencer

## 📊 Monitoring & Status

### Check Sync Progress via SSH

```bash
# Connect to server
ssh user@YOUR_SERVER_IP

# Check service status
sudo systemctl status base-geth base-node --no-pager

# Current block height
curl -s -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' | jq

# Sync status (returns false when fully synced)
curl -s -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' | jq

# Live sync progress in logs
tail -f /DATA/Storage/base-node/logs/node.log | grep "Sync progress"

# Compare with current Base network height
curl -s "https://base.blockscout.com/api/v2/stats" | jq '.total_blocks'
```

### Service Management

```bash
# View logs
tail -20 /DATA/Storage/base-node/logs/geth.log
tail -20 /DATA/Storage/base-node/logs/node.log

# Restart services if needed
sudo systemctl restart base-geth
sudo systemctl restart base-node

# Check system resources
htop
df -h /DATA/Storage
```

## 🔍 Archive Node Capabilities

### Full Historical Access
- **Complete State**: Query any historical block state
- **Transaction History**: Full transaction data since Base genesis
- **Event Logs**: Complete smart contract event history
- **Account Balances**: Historical balance queries at any block

### DeFi Analytics Examples

**1. TVL (Total Value Locked) Analysis**
```bash
# Get historical token balances for major DeFi protocols
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{
    "jsonrpc":"2.0",
    "method":"eth_call",
    "params":[{
      "to":"0x...protocol_contract",
      "data":"0x70a08231000000000000000000000000..."
    }, "0x1B77C20"],"id":1}'
```

**2. Protocol Growth Tracking**
- Track daily active users across Base DeFi protocols
- Monitor liquidity provision patterns
- Analyze yield farming strategies

**3. Cross-Chain Bridge Analysis**
- Ethereum → Base bridging volumes
- Asset flow analysis between L1/L2
- Bridge utilization metrics

**4. Ecosystem Health Metrics**
- Transaction throughput over time
- Gas usage patterns
- Smart contract deployment trends

### Technical Analysis Capabilities

**RPC Methods Available**:
- `eth_getBalance` - Historical balances
- `eth_call` - Smart contract state queries  
- `eth_getTransactionReceipt` - Transaction details
- `eth_getLogs` - Event filtering across blocks
- `debug_traceTransaction` - Transaction execution traces

**Archive-Specific Features**:
- State queries at any historical block height
- Complete contract storage history
- Full transaction trace data
- Historical gas price analysis

## 📈 Expected Sync Timeline

**Current Progress**: ~0% (Block 0 / 29.5M blocks)

**Estimated Completion**:
- **Optimistic**: 1-2 days  
- **Realistic**: 2-4 days
- **Conservative**: 4-7 days

**Factors Affecting Speed**:
- Network bandwidth to Ethereum L1
- Server I/O performance  
- Historical block complexity

## 🚀 Usage Examples

Once synced, the archive node enables powerful analytics:

```python
# Python example - DeFi TVL tracking
import requests

def get_protocol_tvl(contract_addr, block_number):
    payload = {
        "jsonrpc": "2.0",
        "method": "eth_call",
        "params": [{
            "to": contract_addr,
            "data": "0x..."  # encoded function call
        }, hex(block_number)],
        "id": 1
    }
    response = requests.post("http://YOUR_SERVER_IP:8545", json=payload)
    return response.json()

# Analyze TVL growth over 30 days
for day in range(30):
    block = latest_block - (day * 43200)  # ~1 day of blocks
    tvl = get_protocol_tvl("0x...", block)
    print(f"Day {day}: TVL = ${tvl}")
```

## 🔧 Troubleshooting

**If Services Stop**:
```bash
sudo systemctl restart base-geth base-node
sudo journalctl -u base-geth -f
sudo journalctl -u base-node -f
```

**Storage Monitoring**:
```bash
# Check available space (should maintain >20% free)
df -h /DATA/Storage

# Monitor sync database growth
du -sh /DATA/Storage/base-node/data/
```

## 📚 Resources

- **Base Documentation**: https://docs.base.org
- **OP Stack**: https://stack.optimism.io
- **Base Block Explorer**: https://basescan.org
- **Base RPC Methods**: Standard Ethereum JSON-RPC + OP Stack extensions

---

**Setup Completed**: June 25, 2025  
**Node Version**: op-geth v0.1.0, op-node untagged-6540f767  
**Network**: Base Mainnet (Chain ID: 8453)