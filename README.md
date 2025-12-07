# VPS Benchmark Script

A bash script for systematic performance testing of VPS environments. Collects CPU, memory, disk, and network metrics with historical tracking and comparison capabilities.

## Features

- CPU benchmarking (single and multi-threaded via sysbench)
- Memory bandwidth testing
- Disk I/O performance (buffered/direct writes, reads, latency)
- Network speed tests using Ookla Speedtest CLI
- SQLite database for historical data storage
- Comparison mode for tracking performance changes over time
- JSON export for integration with monitoring systems
- Optional ntfy push notifications
- Dependency caching to minimize installation overhead

## Requirements

**Tested on:** Debian 12 (Bookworm), Ubuntu 20.04+

**Dependencies** (auto-installed on first run):

- sysbench
- bc
- sqlite3
- curl
- ioping
- speedtest-cli (Ookla) or speedtest-cli (Python)

Root access required for initial dependency installation only.

## Installation

```bash
git clone https://github.com/buildplan/vps-bm.git
cd vps-benchmark
chmod +x vps-bm.sh
```

## Usage

### Basic benchmark

```bash
./vps-bm.sh
```

### Save results to database

```bash
sudo ./vps-bm.sh --save
```

### Compare with previous run

```bash
sudo ./vps-bm.sh --compare
```

### Quick mode (reduced test times)

```bash
sudo ./vps-bm.sh --quick --compare
```

### Export JSON

```bash
sudo ./vps-bm.sh --json
```

### List historical benchmarks

```bash
./vps-bm.sh --list
```

## Command Line Options

| Option | Description |
| :-- | :-- |
| `-s, --save` | Save results to SQLite database |
| `-c, --compare` | Save and compare with previous benchmark |
| `-l, --list` | List all saved benchmark runs |
| `-q, --quick` | Fast benchmark (5s CPU, 512MB disk) |
| `-j, --json` | Export results as JSON |
| `-h, --help` | Show help message |

## Configuration

Create `.benchmark_config` in the script directory to override defaults:

```bash
# Test durations
CPU_TEST_TIME=10          # seconds
DISK_TEST_SIZE=1024       # MB

# Network testing
SKIP_NETWORK=0            # set to 1 to skip

# ntfy notifications
NTFY_ENABLED=0
NTFY_URL=""
NTFY_TOKEN=""
NTFY_TOPIC="vps-benchmarks"
```

## Output Files

- `benchmark_results.db` - SQLite database with historical data
- `benchmark_latest.json` - Most recent results in JSON format
- `benchmark.log` - Execution log with timestamps
- `.deps_installed` - Dependency verification marker

## Automated Monitoring

Add to crontab for weekly benchmarks:

```bash
# Weekly benchmark every Sunday at 2 AM
0 2 * * 0 /path/to/vps-bm.sh --compare >> /var/log/vps-benchmark-cron.log 2>&1
```

## JSON Output Format

The `benchmark_latest.json` file follows this structure for integration with monitoring systems like Grafana or custom dashboards:

```json
{
  "version": "0.2.0",
  "timestamp": "2025-12-07T16:40:35Z",
  "hostname": "server-name",
  "is_docker": false,
  "metrics": {
    "cpu": {
      "single_thread_events_per_sec": 1437.44,
      "multi_thread_events_per_sec": 5478.04
    },
    "memory": {
      "bandwidth_mib_per_sec": 18709.80
    },
    "disk": {
      "write_buffered_mbs": 839,
      "write_direct_mbs": 573,
      "read_mbs": 1331,
      "latency_us": 771
    },
    "network": {
      "download_mbps": 197.20,
      "upload_mbps": 188.33,
      "latency_ms": 4.37
    }
  }
}
```

## Performance Metrics

**CPU:** Events per second from sysbench prime number calculation
**Memory:** Sequential memory bandwidth in MiB/s
**Disk:** Read/write throughput in MB/s, latency in microseconds
**Network:** Mbps download/upload speeds, ping latency in milliseconds

## Known Limitations

- Ookla speedtest may occasionally report upload failures due to timeout issues - the script continues and captures available metrics
- ioping requires installation on first run but is optional for basic benchmarking
- Network tests require outbound connectivity to speedtest servers
- Some metrics may be affected when running in containerized environments

## Troubleshooting

**"Missing dependencies" error:** Run script with `sudo` on first execution to install required packages

**Upload test fails:** Normal for some network configurations - download and latency metrics are still captured

**Database schema errors:** Delete `benchmark_results.db` to recreate with current schema, or the script will attempt automatic migration

## Contributing

Bug reports and pull requests welcome. For major changes, please open an issue first.

## License

MIT License - see LICENSE file for details
