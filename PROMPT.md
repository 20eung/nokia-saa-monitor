# Project Prompt: Nokia SAA-driven Performance Monitoring System

## Role
You are an expert Network Automation Engineer and Python Developer.

## Goal
Create a performance monitoring system for Nokia network devices (7210 SAS, 7705 SAR, 7750 SR) using SAA (Service Assurance Agent) and ETH-CFM.

## Key Requirements
1. **Data Collection**:
   - Use Python with `Netmiko` for SSH scraping.
   - Support both Classic CLI (TiMOS-B) and MD-CLI (TiMOS-C).
   - Implement multi-threading for concurrent collection.
   - Parse CLI output using `TextFSM` or Regex.
2. **Data Pipeline**:
   - Push collected data to `Telegraf` (HTTP Listener).
   - Store data in `InfluxDB v1.8`.
3. **Visualization**:
   - Use `Grafana` for dashboards.
   - Monitor Latency (RTT), Jitter, and Packet Loss.
4. **Device Optimization**:
   - Respect resource constraints of older devices (e.g., 7210 SAS-M).
   - Use SSH connection pooling.
   - Implement intelligent polling intervals.

## Output Structure
- `collector/`: Python source code for the scraper.
- `templates/`: TextFSM templates for parsing Nokia CLI.
- `config/`: Telegraf and InfluxDB configuration files.
- `dashboards/`: Grafana dashboard JSON exports.
- `README.md`: Project documentation.
