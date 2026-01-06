# Chorus

This repository contains the experimental data and network traces for the paper:

> **Chorus: Coordinating Mobile Multipath Scheduling and Adaptive Video Streaming**
>
> Gerui Lv, Qinghua Wu, Yanmei Liu, Zhenyu Li, Qingyue Tan, Furong Yang, Wentao Chen, Yunfei Ma, Hongyu Guo, Ying Chen, Gaogang Xie
>
> *ACM MobiCom '24, November 18-22, 2024, Washington, D.C., USA*
>
> DOI: [10.1145/3636534.3649359](https://doi.org/10.1145/3636534.3649359)

## Overview

Chorus is a cross-layer framework that coordinates multipath scheduling with adaptive video streaming to jointly optimize Quality of Experience (QoE). Unlike traditional approaches that tune the packet scheduler independently, Chorus establishes two-way feedback control loops between the server transport layer and the client application, introducing **Coarse-grained Decisions (CD)** and **Fine-grained Corrections (FC)** to ensure appropriate bitrate selection and expected-time-oriented transport performance.

### Key Results

- **Emulation**: Chorus improves average QoE by **23.5%** over XLINK (state-of-the-art MPQUIC scheduler)
- **Real-world**: Chorus improves average QoE by **65.7%~114.4%** over XLINK and single-path QUIC

## Code Availability

Due to code ownership restrictions and legal reasons, the Chorus core implementation and server-side integration code cannot be open-sourced at this time. However, the Android client implementation code may be released in the future (pending cleanup and organization).

This repository provides the main experimental datasets to facilitate reproducibility and further research.

## Repository Structure

```
Chorus/
├── bw_traces/                  # Network bandwidth traces for emulation
├── emulation_results/          # Trace-driven emulation results (§5.2)
│   ├── Chorus/                 # Chorus results
│   ├── XLINK/                  # XLINK results
│   ├── MinRTT/                 # MinRTT results
│   ├── MinRTTRI/               # MinRTT+RI results
│   ├── SP/                     # Single-path (SP) results
│   └── *.csv                   # Summary metrics files
└── real-world_results/         # Real-world evaluation results (§5.5)
    ├── 1strong/                # Strong scenario
    ├── 2middle/                # Medium scenario
    └── 3weak/                  # Weak scenario
```

## ABR Schemes

| Scheme | Description |
|:-------|:------------|
| **Chorus** | Proposed cross-layer framework with CD&FC |
| **XLINK** | State-of-the-art QoE-driven MPQUIC scheduler (Baseline) |
| **MinRTT** | Basic MPQUIC scheduler, shortest RTT path selection (Baseline) |
| **MinRTT+RI** | MinRTT with unlimited Reinjection (upper bound of XLINK) |
| **SP** | Single-path QUIC (Baseline) |

## Dataset Details

### Network Traces (`bw_traces/`)

Contains 52 network bandwidth traces collected from real 4G/5G cellular and WiFi networks:

| File Type | Description |
|:----------|:------------|
| `.pps` | Packet-per-second traces for network emulation (Mahimahi/mpshell) |
| `.csv` | Time-series bandwidth data (Timestamp in seconds, Bandwidth in Mbps) |
| `.png` | Bandwidth trace visualizations |
| `all_trace_bw.csv` | Summary of all traces with average bandwidth, RTT, and loss rate |

**Trace naming**: `downlink-{scenario}.csv`
- Cellular scenarios: `cellular_airport`, `cellular_driving`, `cellular_home`, `cellular_hsr` (high-speed rail), `cellular_office`, `cellular_outdoor`, `cellular_subway`, etc.
- WiFi scenarios: `wifi_home`, `wifi_office`, `wifi_walking`, `wifi_hsr`

### Emulation Results (`emulation_results/`)

**Paper Section**: Section 5.2 (Trace-driven Evaluation)

Each algorithm directory contains:
- `{Algorithm}.test` - Test configuration file
- `summary_{Algorithm}.csv` - Per-trace QoE results
- `pred_summary_{Algorithm}.csv` - Prediction metrics per trace
- `tests/` - Raw per-test log files (converted to .csv)

**Test group naming format** (in `tests/` directory):
```
{test_number}~{trace_name_path1}~{owd_path1}~{trace_name_path2}~{owd_path2}
```

Example: `1~cellular_subway_3~25~cellular_airport~35` represents:
- Test #1
- Path 1: `cellular_subway_3` trace with 25ms OWD (one-way delay)
- Path 2: `cellular_airport` trace with 35ms OWD

**`.test` file format**:
```
Line 1: {algorithm} {duration_seconds}
Line 2: {path_type} {cc_count} {congestion_control}
        # MP = multipath, SP = single-path
Lines 3+: {trace1} {owd1} {loss1} {size1} {trace2} {owd2} {loss2} {size2}
```

**Summary metrics files**:
- `overall_emulation_qoe_metrics.csv` - QoE comparison (mean ± confidence interval)
- `overall_emulation_mp_metrics.csv` - Multipath transport metrics
- `overall_emulation_predict_metrics.csv` - Prediction accuracy metrics

### Real-world Results (`real-world_results/`)

**Paper Section**: Section 5.5 (Real-world Evaluation)

108 field tests conducted on Android devices (4G and 5G smartphones) over commercial networks:

| Scenario | Directory | Description |
|:---------|:----------|:------------|
| **Strong** | `1strong/` | WiFi bandwidth > 16 Mbps (highest bitrate) |
| **Medium** | `2middle/` | WiFi < 16 Mbps, but aggregated multipath sufficient |
| **Weak** | `3weak/` | Aggregated bandwidth often < 16 Mbps (critical scenario) |

**Per-test data structure**: `{scenario}/test{N}/{Algorithm}/`
- `{Algorithm}_client_{N}.csv` - Client-side per-chunk metrics
- `{Algorithm}_server_{N}.csv` - Server-side metrics

**Client CSV columns include**:
- `chunk_index`, `quality`, `bitrate`, `buffer_level`
- `delivery_time`, `rebuffer_time`, `QoE`
- Throughput predictions: `pre_throughput`, `hm_throughput`
- Path metrics: `fast_path_bw`, `slow_path_bw`, `fast_path_rtt`, `slow_path_rtt`

## Main Experimental Configuration

| Parameter | Value |
|:----------|:------|
| Video codec | H.264/MPEG-4 |
| Chunk duration | 4 seconds |
| Bitrate ladder | {1, 2.5, 5, 8, 16} Mbps |
| Resolution levels | 360p, 480p, 720p, 1080p, 1440p (2K) |
| ABR algorithm | MPC (Model Predictive Control) |
| Congestion control | Cubic (decoupled per path) |

Other settings are evaluated in §5.3 and §5.4. See the paper for details.

## QoE Metric

Linear QoE formula (following MPC):

```
QoE = Σ Rₖ - μ·Σ Tₖ - λ·Σ |Rₖ₊₁ - Rₖ|
```

Where:

- `Rₖ`: Bitrate of chunk k (Mbps)
- `Tₖ`: Rebuffering time of chunk k (seconds)
- `μ = 16`: Rebuffering penalty coefficient (corresponding to the highest bitrate)
- `λ = 1`: Bitrate switch penalty coefficient

## Citation

If you use this dataset in your research, please cite:

```bibtex
@inproceedings{lv2024chorus,
  title={Chorus: Coordinating Mobile Multipath Scheduling and Adaptive Video Streaming},
  author={Lv, Gerui and Wu, Qinghua and Liu, Yanmei and Li, Zhenyu and Tan, Qingyue and Yang, Furong and Chen, Wentao and Ma, Yunfei and Guo, Hongyu and Chen, Ying and Xie, Gaogang},
  booktitle={Proceedings of the 30th Annual International Conference on Mobile Computing and Networking (MobiCom '24)},
  pages={246--262},
  year={2024},
  publisher={ACM},
  address={New York, NY, USA},
  doi={10.1145/3636534.3649359}
}
```

## Related Resources

- **XQUIC**: [https://github.com/alibaba/xquic](https://github.com/alibaba/xquic) - User-space QUIC library used for Chorus implementation
- **Mahimahi**: [https://github.com/ravinet/mahimahi](https://github.com/ravinet/mahimahi) - Network emulation tool for trace replay
- **mpshell**: [https://github.com/ravinet/mahimahi/tree/old/mpshell_scripted](https://github.com/ravinet/mahimahi/tree/old/mpshell_scripted) - Multipath version of Mahimahi

## License

This dataset is released under the [Creative Commons Attribution 4.0 International License (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).