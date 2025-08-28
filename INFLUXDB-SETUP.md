# InfluxDB Setup and Integration Guide

This guide covers setting up InfluxDB for the IoT Tank Level Monitoring Dashboard to store and query historical sensor data.

## Prerequisites

- InfluxDB 2.x installed and running
- Node.js and npm/yarn for dependency installation
- Access to InfluxDB web interface (default: http://localhost:8086)

## 1. InfluxDB Installation

### Option A: Docker (Recommended)
```bash
# Pull and run InfluxDB 2.x
docker run -d \
  --name influxdb \
  -p 8086:8086 \
  -v influxdb-storage:/var/lib/influxdb2 \
  -v influxdb-config:/etc/influxdb2 \
  influxdb:2.7
```

### Option B: Direct Installation
Download from [InfluxData Downloads](https://portal.influxdata.com/downloads/) and follow platform-specific instructions.

## 2. Initial InfluxDB Configuration

1. **Access InfluxDB UI**: Navigate to http://localhost:8086
2. **Complete Setup Wizard**:
   - Username: `admin`
   - Password: `your-secure-password`
   - Organization: `iot-monitoring`
   - Initial Bucket: `sensor-data`

3. **Generate Admin Token**:
   - Go to Data → API Tokens
   - Click "Generate API Token" → "All Access API Token"
   - Copy the token for environment configuration

## 3. Create Required Buckets

```bash
# Using InfluxDB CLI
influx bucket create \
  --name sensor-data \
  --org iot-monitoring \
  --retention 30d

influx bucket create \
  --name sensor-data-hourly \
  --org iot-monitoring \
  --retention 1y

influx bucket create \
  --name sensor-data-daily \
  --org iot-monitoring \
  --retention 5y
```

## 4. Environment Configuration

Create `.env` file in project root:

```bash
# InfluxDB Configuration
VITE_INFLUX_URL=http://localhost:8086
VITE_INFLUX_TOKEN=your-generated-admin-token-here
VITE_INFLUX_ORG=iot-monitoring
VITE_INFLUX_BUCKET=sensor-data

# WebSocket Configuration
VITE_WEBSOCKET_URL=ws://localhost:1880/ws/data

# MQTT Configuration  
VITE_MQTT_URL=ws://localhost:9001
VITE_MQTT_TOPIC=Sensor/data
```

## 5. Install Dependencies

```bash
npm install @influxdata/influxdb-client
```

## 6. Data Schema

### Tank Data Points
- **Measurement**: `tank_levels`
- **Tags**: `tank_id`, `tank_name`, `location`, `status`, `liquid_type`
- **Fields**: `level_percentage`, `current_volume_m3`, `available_volume_m3`, `capacity_liters`, `percentage_full`, `temperature_celsius`

### Flow Meter Data Points
- **Measurement**: `flow_meters`
- **Tags**: `meter_id`, `meter_name`, `location`, `status`, `category`, `direction`
- **Fields**: `flow_rate_lpm`, `total_volume_liters`, `pressure_bar`

## 7. Data Retention Policies

### Raw Data (30 days)
```flux
option task = {name: "downsample-hourly", every: 1h}

from(bucket: "sensor-data")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "tank_levels" or r._measurement == "flow_meters")
  |> aggregateWindow(every: 1h, fn: mean, createEmpty: false)
  |> to(bucket: "sensor-data-hourly")
```

### Hourly Aggregates (1 year)
```flux
option task = {name: "downsample-daily", every: 1d}

from(bucket: "sensor-data-hourly")
  |> range(start: -1d)
  |> aggregateWindow(every: 1d, fn: mean, createEmpty: false)
  |> to(bucket: "sensor-data-daily")
```

## 8. Common Queries

### Tank Level History (24 hours)
```flux
from(bucket: "sensor-data")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "tank_levels")
  |> filter(fn: (r) => r.tank_id == "tank-001")
  |> filter(fn: (r) => r._field == "level_percentage")
```

### Flow Rate Trends (7 days)
```flux
from(bucket: "sensor-data-hourly")
  |> range(start: -7d)
  |> filter(fn: (r) => r._measurement == "flow_meters")
  |> filter(fn: (r) => r._field == "flow_rate_lpm")
  |> aggregateWindow(every: 1h, fn: mean)
```

### System Statistics
```flux
from(bucket: "sensor-data")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "tank_levels")
  |> group(columns: ["tank_id"])
  |> count()
  |> group()
  |> sum()
```

## 9. Integration Features

### Automatic Data Storage
- Real-time sensor data automatically stored via `dataProcessingService`
- All derived calculations (volume in m³, status, etc.) preserved
- Error handling prevents data loss from breaking real-time flow

### Historical Analytics
- `getTankHistory()` - Tank level trends
- `getFlowMeterHistory()` - Flow rate analysis  
- `getTankAnalytics()` - Aggregated statistics
- `getSystemStats()` - System-wide metrics

### Performance Optimization
- Batch writes for efficiency
- Connection pooling
- Configurable retention policies
- Automatic downsampling

## 10. Monitoring and Maintenance

### Health Checks
```bash
# Check InfluxDB status
curl -I http://localhost:8086/health

# Verify bucket exists
influx bucket list --org iot-monitoring
```

### Backup Strategy
```bash
# Backup all data
influx backup /path/to/backup --org iot-monitoring

# Restore from backup
influx restore /path/to/backup --org iot-monitoring
```

## 11. Troubleshooting

### Connection Issues
- Verify InfluxDB is running: `docker ps` or service status
- Check firewall settings for port 8086
- Validate token permissions in InfluxDB UI

### Data Not Appearing
- Check browser console for InfluxDB client errors
- Verify environment variables are loaded correctly
- Ensure bucket exists and token has write permissions

### Performance Issues
- Monitor bucket size and implement retention policies
- Use continuous queries for large datasets
- Consider InfluxDB clustering for high-volume scenarios

## 12. Security Considerations

- Use read-only tokens for dashboard queries
- Implement token rotation policy
- Enable HTTPS in production
- Configure proper network access controls
- Regular backup and disaster recovery testing

This setup provides a robust, scalable database solution for your IoT monitoring system using only real sensor data and derived calculations.
