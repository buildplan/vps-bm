# VPS Benchmark Script

A bash script for systematic performance testing of VPS environments. Collects CPU, memory, disk, and network metrics with historical tracking and comparison capabilities.

## Features

- CPU benchmarking (single and multi-threaded via sysbench)
- Memory bandwidth testing
- Disk I/O performance using FIO (flexible I/O tester)
  - Sequential read/write tests with direct and buffered I/O
  - Automatic ioengine detection (libaio/sync)
  - Latency measurements via ioping
- Network speed tests using Ookla Speedtest CLI or Python speedtest-cli
- SQLite database for historical data storage
- Comparison mode for tracking performance changes over time
- JSON export for integration with monitoring systems
- Optional ntfy push notifications
- Dependency caching to minimize installation overhead
- System health checks (disk usage, filesystem type, load average)

## Requirements

**Tested on:** Debian 12 (Bookworm), Ubuntu 20.04+, Fedora, RHEL/CentOS

**Dependencies** (auto-installed on first run):

- sysbench
- fio
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
| `-q, --quick` | Fast benchmark (5s CPU, 256MB disk) |
| `-j, --json` | Export results as JSON |
| `-h, --help` | Show help message |

## Configuration

Create `.benchmark_config` in the script directory to override defaults:

```bash
# Test durations and sizes
CPU_TEST_TIME=10          # seconds (default: 10)
DISK_TEST_SIZE="1G"       # FIO size format (default: 1G)

# Network testing
SKIP_NETWORK=0            # set to 1 to skip network tests
SPEEDTEST_SERVER_ID=""    # specific server ID, or empty for auto-select

# ntfy notifications
NTFY_ENABLED=0
NTFY_URL="https://ntfy.sh"
NTFY_TOKEN=""
NTFY_TOPIC="vps-bm"
```

### Speedtest Server Selection

To use a specific speedtest server for consistent testing:

1. Find server ID: `speedtest --servers`
2. Set in config: `SPEEDTEST_SERVER_ID="12345"`

This ensures network tests always use the same endpoint for accurate historical comparisons.

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

The `benchmark_latest.json` file follows this structure for integration with monitoring systems:

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

## Performance Metrics Explained

### CPU

Events per second from sysbench prime number calculation (higher = better)

### Memory

Sequential memory bandwidth in MiB/s from sysbench memory test

### Disk (FIO)

- **Write (Buffered)**: Sequential write with OS cache (simulates typical file operations)
- **Write (Direct)**: Direct I/O write bypassing cache (measures raw disk performance)
- **Read (Direct)**: Direct I/O read bypassing cache
- **Latency**: Average I/O latency in microseconds from ioping (lower = better)

FIO automatically selects the best I/O engine (libaio on most systems, fallback to sync).

### Network

- **Download/Upload**: Mbps measured via Ookla Speedtest or Python speedtest-cli
- **Latency**: Ping latency in milliseconds

## System Health Checks

The script performs pre-flight checks before benchmarking:

- **Disk Usage**: Warns if >90% full
- **Filesystem Type**: Detects tmpfs (RAM disk) and warns that disk benchmarks will be invalid
- **Load Average**: Reports current system load per CPU
- **Docker Detection**: Identifies containerized environments

If running from `/tmp` or a RAM disk, move the script to a physical disk location like `/root` or `/home` for accurate disk benchmarks.

## Example Output

```text
=== FINAL RESULTS SUMMARY ===

CPU Performance (sysbench):
  Single-Thread        [✓]:  1500.57 events/sec
  Multi-Thread         [✓]:  5669.13 events/sec

Memory Performance:
  Bandwidth            [✓]: 20026.91 MiB/s

Disk Performance (FIO):
  Write (Buffered)     [✓]: 807 MB/s
  Write (Direct)       [✓]: 570 MB/s
  Read (Direct)        [✓]: 1536 MB/s
  Latency              [✓]: 771 μs

Network Performance (speedtest):
  Download             [✓]: 197.20 Mbps
  Upload               [✓]: 187.74 Mbps
  Latency              [✓]: 4.37 ms

=== COMPARISON WITH PREVIOUS RUN ===
Previous Run: 2025-12-07 16:40:35 (v0.2.0)

CPU Performance:
  Single-Thread (ev/s)     : 1437.44 →  1500.57 (▲4.0%)
  Multi-Thread (ev/s)      : 5478.04 →  5669.13 (▲3.0%)

Disk Performance (MB/s):
  Write Buffered           : 839 → 807 (▼3.0%)
  Read Direct              : 1331 → 1536 (▲15.0%)
```

## Known Limitations

- Network upload tests may occasionally timeout on congested connections - download and latency metrics are still captured
- Disk benchmarks require sufficient free space (default 1GB for tests)
- Some VPS providers limit or throttle I/O operations which may affect results
- Container environments (Docker, LXC) may show different results than bare metal due to virtualization overhead

## Troubleshooting

**"Missing dependencies" error**
Run with `sudo` on first execution to install required packages.

**"Running in TMPFS" warning**
Script is in a RAM disk (`/tmp` or similar). Move to persistent storage: `mv vps-bm.sh /root/`

**FIO "libaio not supported" message**
Normal on some systems. Script automatically falls back to sync engine.

**Speedtest upload fails**
Common on rate-limited connections. Download and latency metrics are still recorded.

**"No space left on device"**
Reduce `DISK_TEST_SIZE` in config (e.g., `DISK_TEST_SIZE="512M"`).

## Why FIO Instead of dd?

FIO (Flexible I/O Tester) provides more accurate and consistent results than dd:

- Better control over I/O patterns and direct I/O
- Less affected by OS caching
- Industry-standard tool for storage benchmarking
- More realistic simulation of real-world workloads

## Contributing

Bug reports and pull requests are welcome. For major changes, open an issue first to discuss proposed modifications.

## License

MIT License - see LICENSE file for details
