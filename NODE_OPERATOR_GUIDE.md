# Base Node Operator Guide

A practical guide for operating and maintaining a Base node for production environments.

## Table of Contents

- [Pre-Deployment Checklist](#pre-deployment-checklist)
- [Network Selection](#network-selection)
- [Client Selection](#client-selection)
- [Hardware Optimization](#hardware-optimization)
- [Monitoring & Health Checks](#monitoring--health-checks)
- [Maintenance & Updates](#maintenance--updates)
- [Performance Tuning](#performance-tuning)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)
- [Backup & Recovery](#backup--recovery)

## Pre-Deployment Checklist

### Infrastructure Requirements

- [ ] Verify minimum system specifications (32GB RAM, 2TB NVMe SSD)
- [ ] Test L1 node RPC endpoints for reliability
- [ ] Confirm Docker and Docker Compose installation
- [ ] Verify network connectivity and firewall rules
- [ ] Allocate sufficient disk space (account for growth)
- [ ] Set up monitoring infrastructure
- [ ] Configure backup procedures
- [ ] Document hardware configuration and network setup

### Network Preparation

- [ ] Test L1 ETH RPC endpoint connectivity
  ```bash
  curl -X POST http://your-l1-rpc:8545 \
    -H "Content-Type: application/json" \
    -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
  ```
- [ ] Test L1 Beacon endpoint connectivity
  ```bash
  curl http://your-l1-beacon:3500/eth/v1/node/health
  ```
- [ ] Verify bandwidth availability (recommend 100+ Mbps)
- [ ] Test latency to L1 endpoints

## Network Selection

### Mainnet vs Testnet

| Aspect | Mainnet | Testnet (Sepolia) |
|--------|---------|------------------|
| Environment File | `.env.mainnet` | `.env.sepolia` |
| Use Case | Production | Development/Testing |
| Disk Space | ~2-3TB | ~500GB |
| Sync Time | 2-7 days | 1-3 days |
| Transaction Fees | Real ETH | Free Sepolia ETH |

## Client Selection

### Reth (Recommended)

**Pros:**
- Written in Rust for better performance
- Lower memory footprint
- Excellent sync speed
- Good documentation

**Startup:**
```bash
docker compose up --build
```

### Geth

**Pros:**
- Battle-tested, widely used
- Excellent documentation
- Good community support

**Startup:**
```bash
CLIENT=geth docker compose up --build
```

### Nethermind

**Pros:**
- Full-featured with JSON-RPC extensions
- Good for archive node operations
- Enterprise support available

**Startup:**
```bash
CLIENT=nethermind docker compose up --build
```

## Hardware Optimization

### Storage Configuration

**SSD Setup:**
- Use NVMe drives for best performance
- Consider RAID 0 for multiple drives (performance over redundancy)
- Format with ext4 filesystem
- Monitor disk I/O and temperature

**LVM Configuration (Advanced):**
```bash
# Create physical volume
sudo pvcreate /dev/nvme0n1

# Create volume group
sudo vgcreate base_vg /dev/nvme0n1

# Create logical volume
sudo lvcreate -l 100%FREE -n base_lv base_vg

# Format
sudo mkfs.ext4 /dev/base_vg/base_lv
```

### Memory & CPU Tuning

**Cache Configuration (Geth):**
```bash
# In your .env file
GETH_CACHE=20480          # 20GB (adjust based on RAM)
GETH_CACHE_DATABASE=20    # 4GB
GETH_CACHE_GC=12
GETH_CACHE_SNAPSHOT=24
GETH_CACHE_TRIE=44
```

**System Tuning:**
```bash
# Increase file descriptors
echo "fs.file-max = 2097152" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Network tuning
echo "net.ipv4.tcp_max_syn_backlog = 4096" | sudo tee -a /etc/sysctl.conf
echo "net.core.somaxconn = 4096" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Monitoring & Health Checks

### Key Metrics to Monitor

1. **Sync Status**
   ```bash
   curl http://localhost:8545 -X POST \
     -H "Content-Type: application/json" \
     -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}'
   ```

2. **Peer Count**
   ```bash
   curl http://localhost:8545 -X POST \
     -H "Content-Type: application/json" \
     -d '{"jsonrpc":"2.0","method":"net_peerCount","params":[],"id":1}'
   ```

3. **Block Height**
   ```bash
   curl http://localhost:8545 -X POST \
     -H "Content-Type: application/json" \
     -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
   ```

### Docker Monitoring

```bash
# Check container logs
docker compose logs -f

# Monitor resource usage
docker stats

# Check container health
docker compose ps
```

### Grafana Dashboard Setup

1. Configure Prometheus to scrape your node metrics
2. Import Base Node dashboard templates
3. Set up alerts for:
   - Sync lag > 5 blocks
   - Peer count < 5
   - Memory usage > 90%
   - Disk space < 10%

## Maintenance & Updates

### Regular Maintenance Tasks

**Daily:**
- Monitor sync status
- Check peer connectivity
- Verify disk space
- Review error logs

**Weekly:**
- Run full backup
- Analyze performance metrics
- Check for updates
- Verify RPC endpoints

**Monthly:**
- Update Docker images
- Review and optimize cache settings
- Analyze storage growth
- Test recovery procedures

### Version Updates

```bash
# Stop current node
docker compose down

# Pull latest images
docker compose pull

# Start updated node
docker compose up --build -d

# Monitor startup and sync
docker compose logs -f
```

## Performance Tuning

### Optimizing Sync Speed

1. **Increase resource allocation:**
   ```bash
   GETH_CACHE=24576 docker compose up --build
   ```

2. **Use snapshot sync (if available):**
   - Download Base snapshots from official sources
   - Extract to data directory before starting
   - Significantly reduces initial sync time

3. **Optimize L1 RPC:**
   - Use direct connection instead of public RPC
   - Consider multiple L1 endpoints with fallback
   - Monitor L1 RPC latency

### Query Performance

- Enable `archive` mode if historical data needed
- Use `full` sync mode for most use cases
- Implement connection pooling in applications
- Cache frequently accessed data

## Troubleshooting

### Node Won't Sync

**Symptoms:** Block height not increasing

**Troubleshooting:**
1. Verify L1 RPC connectivity
2. Check peer count: `net_peerCount`
3. Review logs for errors
4. Ensure sufficient disk space
5. Restart container: `docker compose restart`

### Out of Memory

**Solution:**
```bash
# Reduce cache sizes in your .env
GETH_CACHE=10240
GETH_CACHE_DATABASE=10
```

### Disk Space Issues

```bash
# Check disk usage
df -h /path/to/data

# Prune old data (if supported)
docker compose exec geth geth --http.addr 0.0.0.0 db compact
```

### High Latency

1. Check network bandwidth
2. Monitor CPU usage
3. Check disk I/O
4. Review peer quality
5. Verify L1 RPC latency

## Security Considerations

### Network Security

- Restrict RPC access to known IPs
- Use firewall rules to limit port exposure
- Consider using reverse proxy with rate limiting
- Enable TLS/SSL for remote connections

### Access Control

```bash
# Run node with limited RPC methods
docker compose.yml configuration:
OP_NODE_RPC_LISTEN_ADDR=127.0.0.1  # Only local access
```

### Data Protection

- Encrypt data at rest (if sensitive)
- Regular backups to secure storage
- Monitor access logs
- Keep Docker images updated

## Backup & Recovery

### Backup Strategy

```bash
# Full backup
tar -czf base-node-backup-$(date +%Y%m%d).tar.gz /path/to/data

# Incremental backup
rsync -av /path/to/data /backup/location
```

### Recovery Procedure

1. Stop the node: `docker compose down`
2. Clear data directory: `rm -rf /path/to/data/*`
3. Restore from backup: `tar -xzf backup-file.tar.gz`
4. Verify data integrity
5. Start node: `docker compose up --build`

### Snapshot Recovery

1. Download latest Base snapshot
2. Extract to data directory
3. Verify checksum
4. Start node (will resume sync from snapshot)

## Additional Resources

- [Base Docs - Run a Node](https://docs.base.org/chain/run-a-base-node)
- [Optimism Stack Documentation](https://docs.optimism.io/)
- [Reth GitHub](https://github.com/paradigmxyz/reth)
- [Geth Documentation](https://geth.ethereum.org/docs)
- [Base Discord - Node Operators](https://base.org/discord)

## Support

For operational issues:
1. Check the [troubleshooting section](#troubleshooting)
2. Review logs: `docker compose logs`
3. Join Base Discord #node-operators channel
4. Open an issue on GitHub if you find a bug

## Disclaimer

Running a Base node is at your own risk. Monitor your node regularly and keep software updated. Always have backups.
