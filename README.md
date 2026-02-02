# vPLC Real-Time Benchmark

Reproducibility artifacts for the paper: **"Optimization and Evaluation of Container-based EtherCAT Communication in Virtualized PLCs"** submitted to IEEE Transactions on Industrial Electronics.

This repository provides kernel configurations, container configurations, and raw measurement logs to enable reproduction of the experimental results presented in the paper.

## Repository Structure

```
vplc-realtime-benchmark/
├── platform1/                          # ARM64 (PhytiumPi E2000Q)
│   ├── Kernel Configurations/
│   │   ├── cmdline.txt                 # Kernel boot parameters
│   │   └── config.gz                   # PREEMPT_RT kernel config
│   ├── Container Configurations/
│   │   ├── Dockerfile                  # Container image definition
│   │   └── docker-compose.yml          # Container runtime configuration
│   └── Raw measurement (with stress) logs/
│       ├── generic_1000/               # Generic mode, 1000μs cycle
│       ├── generic_125/                # Generic mode, 125μs cycle
│       ├── opt.uio_1000/               # Optimized UIO mode, 1000μs cycle
│       ├── opt.uio_125/                # Optimized UIO mode, 125μs cycle
│       ├── igh_1000/                   # IgH EtherCAT Master, 1000μs cycle
│       └── igh_125/                    # IgH EtherCAT Master, 125μs cycle
├── platform2/                          # x86_64 (Intel Core i5-1135G7)
│   ├── Kernel Configurations/
│   │   ├── cmdline.txt                 # Kernel boot parameters
│   │   └── config-6.16.3-rt3           # PREEMPT_RT kernel config
│   ├── Container Configurations/
│   │   ├── Dockerfile
│   │   └── docker-compose.yml
│   └── Raw measurement (with stress) logs/
└── platform3/                          # RISC-V (StarFive VisionFive 2)
    ├── Kernel Configurations/
    │   ├── cmdline.txt                 # Kernel boot parameters
    │   └── config-6.12.5-rt            # PREEMPT_RT kernel config
    ├── Container Configurations/
    │   ├── Dockerfile
    │   └── docker-compose.yml
    └── Raw measurement (with stress) logs/
```

## Platform Specifications

| Platform | Architecture | CPU | Memory | Kernel |
|----------|-------------|-----|--------|--------|
| Platform 1 | ARM64 | Phytium E2000Q (2×FTC664+2×FTC310) | 4 GB DDR4 | 6.6.63-rt46 |
| Platform 2 | x86_64 | Intel Core i5-12400 (6-core) | 8 GB DDR4 | 6.16.3-rt3 |
| Platform 3 | RISC-V | StarFive JH7110 (4-core U74) | 8 GB LPDDR4 | 6.12.5-rt |

## Kernel Configuration

### Key Boot Parameters

All platforms use similar real-time optimizations:

| Parameter | Description |
|-----------|-------------|
| `isolcpus=managed_irq,domain,N-M` | Isolate CPUs N-M from scheduler and IRQ balancing |
| `nohz_full=N-M` | Disable timer ticks on isolated CPUs |
| `rcu_nocbs=N-M` | Offload RCU callbacks from isolated CPUs |
| `rcu_nocb_poll` | Use polling for RCU callbacks |
| `irqaffinity=0` | Pin IRQs to CPU 0 |
| `nosoftlockup` | Disable soft lockup detector |
| `kthread_cpus=0` | Restrict kernel threads to CPU 0 |

### Platform-Specific Parameters

**x86_64 (Platform 2):**
- `intel_pstate=disable` - Disable Intel P-state driver
- `processor.max_cstate=0` - Disable CPU C-states
- `intel_idle.max_cstate=0` - Disable Intel idle driver
- `intel_iommu=on iommu=pt` - Enable IOMMU in passthrough mode
- `clocksource=tsc tsc=reliable` - Use TSC as clock source
- `hugepages=1024` - Pre-allocate 1024 hugepages

### Runtime Configuration (sysctl)

For optimized UIO mode with `NOHZ_FULL`, the following sysctl settings are required:

```bash
# Allow real-time tasks to use 100% CPU without throttling
sudo sysctl -w kernel.sched_rt_runtime_us=-1

# Disable timer migration for better isolation
sudo sysctl -w kernel.timer_migration=0
```

### Generic Mode Network Configuration

For Generic (socket-based) mode, the NIC must be released from NetworkManager and optimized:

```bash
# Release NIC from NetworkManager
sudo nmcli dev set enp1s0f0 managed no

# Disable IPv6 on the EtherCAT interface
sudo sysctl -w net.ipv6.conf.enp1s0f0.disable_ipv6=1

# Reduce NIC queues to single queue for deterministic behavior
sudo ethtool -L enp1s0f0 combined 1
```

### x86 Platform Additional Configuration

On x86 platforms, disable firmware update timer to prevent interference:

```bash
sudo systemctl stop fwupd-refresh.timer
sudo systemctl disable fwupd-refresh.timer
```

## Container Configuration

### Docker Compose Settings

```yaml
services:
  vecat:
    privileged: true
    cap_add:
      - SYS_NICE      # Required for real-time scheduling
      - IPC_LOCK      # Required for memory locking
    devices:
      - /dev/uio0     # UIO device access
    ulimits:
      rtprio: 99      # Maximum real-time priority
      rttime: -1      # Unlimited real-time CPU time
      memlock: -1     # Unlimited locked memory
    network_mode: host
    volumes:
      - /dev/hugepages:/dev/hugepages
```

### Building the Container

```bash
cd platform1/Container\ Configurations/
docker build -t vecat-benchmark .
```

### Running the Container

```bash
docker-compose up -d
docker exec -it vecat /bin/bash
```

## Raw Measurement Logs

The raw measurement logs (pcapng files) are not included in this repository due to their large size (several GB each). These files are available upon request for researchers who wish to reproduce or verify our results.

**To request measurement logs, please:**
1. Open an issue in this repository describing your research purpose
2. Specify which platform(s) and configuration(s) you need
3. We will provide a download link or transfer the data directly

### Log File Structure

The measurement logs are organized as follows:
- `generic_*` - vECAT Generic mode (socket-based)
- `opt.uio_*` - vECAT Optimized UIO mode (DPDK-based)
- `igh_*` - IgH EtherCAT Master

Each directory contains pcapng files captured during experiments with stress-ng running concurrently.

### Analyzing PCAP Files

Extract jitter statistics using tshark:

```bash
tshark -r capture.pcapng -T fields -e frame.time_delta_displayed | \
  awk '{sum+=$1; sumsq+=$1*$1; if($1>max)max=$1} END {print "Mean:", sum/NR*1e6, "us, Max:", max*1e6, "us"}'
```

## Stress Test Configuration

During measurements, stress-ng was executed with the following parameters:

```bash
stress-ng --cpu 1 --cpu-load 100 --matrix 4 \
          --vm 2 \
          --hdd 2 \
          --iomix 2 \
          --cache 2 \
          ----matrix-3d 4
```

- Real-time threads (vECAT): Isolated cores (e.g., cores 2-3)
- Stress threads: Non-isolated cores (e.g., cores 0,1,4-7)

## Reproducing Results

1. **Prepare the host system:**
   - Install a PREEMPT_RT patched kernel
   - Configure kernel boot parameters as shown in `cmdline.txt`
   - Reboot the system

2. **Set up CPU isolation:**
   ```bash
   # Verify CPU isolation
   cat /sys/devices/system/cpu/isolated
   ```

3. **Configure hugepages:**
   ```bash
   echo 1024 > /proc/sys/vm/nr_hugepages
   mkdir -p /dev/hugepages
   mount -t hugetlbfs nodev /dev/hugepages
   ```

4. **Build and run the container:**
   ```bash
   docker-compose up -d
   ```

5. **Run measurements:**
   - Configure EtherCAT network with sub-devices
   - Start vECAT or IgH master
   - Capture traffic using tcpdump or Wireshark

## Citation

If you use these artifacts in your research, please cite:

```bibtex
@article{vecat2025,
  title={Optimization and Evaluation of Container-based EtherCAT Communication in Virtualized PLCs},
  author={[Authors]},
  journal={IEEE Transactions on Industrial Electronics},
  year={2025},
  note={Under Review}
}
```

## Related Work

The motion control workload used in experiments is based on:

```bibtex
@article{zhou2022hybrid,
  author={Zhou, Nan and Li, Di},
  journal={IEEE Transactions on Industrial Informatics},
  title={Hybrid Synchronous–Asynchronous Execution of Reconfigurable PLC Programs in Edge Computing},
  year={2022},
  volume={18},
  number={3},
  pages={1663-1673},
  doi={10.1109/TII.2021.3092741}
}
```

## License

This repository is provided for academic and research purposes. The measurement data and configuration files are released under the [MIT License](LICENSE).

## Contact

For questions regarding these artifacts, please open an issue in this repository.
