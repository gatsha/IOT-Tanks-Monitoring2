# Node-RED Integration Setup Guide

## Overview
This guide explains how to set up Node-RED to work with your IoT Tank Level Monitoring Dashboard. Node-RED will handle:
- IoT sensor data ingestion via MQTT
- Data processing and validation
- Database storage (InfluxDB)
- Real-time WebSocket updates
- REST API endpoints for the dashboard

## Prerequisites
- Node.js (v16 or higher)
- Node-RED installed globally
- MQTT Broker (Mosquitto recommended)
- InfluxDB (optional, for historical data)

## Installation Steps

### 1. Install Node-RED
```bash
npm install -g node-red
```

### 2. Install Required Node-RED Nodes
```bash
cd ~/.node-red
npm install node-red-dashboard
npm install node-red-contrib-influxdb
npm install node-red-contrib-mqtt-broker
npm install node-red-contrib-websocket
```

### 3. Start Node-RED
```bash
node-red
```

Access the Node-RED editor at: `http://localhost:1880`

## Import the Flow

### 1. Open Node-RED Editor
- Navigate to `http://localhost:1880`
- Click the menu button (☰) in the top-right corner
- Select "Import"

### 2. Import the Flow
- Copy the contents of `node-red-flows.json`
- Paste into the import dialog
- Click "Import"

### 3. Configure Nodes
After importing, you'll need to configure:

#### MQTT Broker Configuration
- Double-click the MQTT Broker node
- Set broker address (usually `localhost`)
- Set port (usually `1883`)
- Configure authentication if needed

#### InfluxDB Configuration
- Double-click the InfluxDB node
- Set hostname (usually `localhost`)
- Set port (usually `8086`)
- Set database name (e.g., `iot_tanks`)
- Configure authentication if needed

#### WebSocket Configuration
- Double-click the WebSocket node
- Set path to `/ws/dashboard`
- Ensure "Whole Message" is unchecked

## MQTT Topics Structure

### Tank Sensors
```
sensors/tank1
{
  "level": 75.5,
  "temperature": 22.3,
  "pressure": 28.1,
  "flowRate": 12.5,
  "sensorId": "TANK001"
}
```

### Flow Meters
```
sensors/flowmeter1
{
  "flowRate": 15.2,
  "totalFlow": 1250.8,
  "pressure": 32.4,
  "temperature": 21.8,
  "status": "active",
  "sensorId": "FLOW001"
}
```

## API Endpoints

### GET /api/tanks
Returns all tank data:
```json
{
  "data": [
    {
      "id": "tank-1",
      "name": "Tank 1",
      "level": 75.5,
      "capacity": 5000,
      "temperature": 22.3,
      "pressure": 28.1,
      "flowRate": 12.5,
      "liquidType": "water",
      "lastUpdated": "2024-01-15T10:30:00.000Z",
      "status": "normal"
    }
  ],
  "status": "success",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

### GET /api/flow-meters
Returns all flow meter data:
```json
{
  "data": [
    {
      "id": "flow-meter-1",
      "name": "Flow Meter 1",
      "flowRate": 15.2,
      "totalFlow": 1250.8,
      "pressure": 32.4,
      "temperature": 21.8,
      "status": "active",
      "lastUpdated": "2024-01-15T10:30:00.000Z"
    }
  ],
  "status": "success",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

### GET /api/system/status
Returns system status:
```json
{
  "data": {
    "overallStatus": "online",
    "activeSensors": 5,
    "totalSensors": 5,
    "lastHeartbeat": "2024-01-15T10:30:00.000Z",
    "alerts": []
  },
  "status": "success",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

## WebSocket Messages

### Tank Updates
```json
{
  "type": "tank_update",
  "payload": {
    "id": "tank-1",
    "level": 75.5,
    "temperature": 22.3,
    "pressure": 28.1,
    "flowRate": 12.5,
    "status": "normal"
  },
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

### Flow Meter Updates
```json
{
  "type": "flow_meter_update",
  "payload": {
    "id": "flow-meter-1",
    "flowRate": 15.2,
    "totalFlow": 1250.8,
    "pressure": 32.4,
    "temperature": 21.8,
    "status": "active"
  },
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

### System Status Updates
```json
{
  "type": "system_status_update",
  "payload": {
    "overallStatus": "online",
    "activeSensors": 5,
    "totalSensors": 5,
    "lastHeartbeat": "2024-01-15T10:30:00.000Z",
    "alerts": []
  },
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

### Heartbeat
```json
{
  "type": "heartbeat",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

## Testing the Integration

### 1. Test MQTT Messages
Use an MQTT client (like MQTT Explorer) to publish test messages:

```bash
# Test tank data
mosquitto_pub -h localhost -t "sensors/tank1" -m '{"level": 75.5, "temperature": 22.3, "pressure": 28.1, "flowRate": 12.5, "sensorId": "TANK001"}'

# Test flow meter data
mosquitto_pub -h localhost -t "sensors/flowmeter1" -m '{"flowRate": 15.2, "totalFlow": 1250.8, "pressure": 32.4, "temperature": 21.8, "status": "active", "sensorId": "FLOW001"}'
```

### 2. Test API Endpoints
```bash
# Test tanks endpoint
curl http://localhost:1880/api/tanks

# Test flow meters endpoint
curl http://localhost:1880/api/flow-meters

# Test system status endpoint
curl http://localhost:1880/api/system/status
```

### 3. Test WebSocket Connection
Use a WebSocket client to connect to `ws://localhost:1880/ws/dashboard`

## Troubleshooting

### Common Issues

#### 1. MQTT Connection Failed
- Check if MQTT broker is running
- Verify broker address and port
- Check authentication credentials

#### 2. InfluxDB Connection Failed
- Ensure InfluxDB is running
- Verify database exists
- Check authentication credentials

#### 3. WebSocket Not Working
- Check WebSocket node configuration
- Verify path is correct
- Check browser console for errors

#### 4. API Endpoints Not Responding
- Ensure HTTP nodes are properly configured
- Check Node-RED logs for errors
- Verify node connections

### Debug Mode
Enable debug mode in Node-RED:
1. Go to Node-RED settings
2. Enable "Debug" mode
3. Check debug panel for detailed logs

## Security Considerations

### 1. MQTT Security
- Use TLS/SSL for MQTT connections
- Implement authentication
- Use strong passwords

### 2. API Security
- Implement API key authentication
- Use HTTPS in production
- Rate limiting

### 3. WebSocket Security
- Implement authentication
- Use WSS in production
- Validate message format

## Production Deployment

### 1. Environment Variables
Set production environment variables:
```bash
export NODE_ENV=production
export MQTT_BROKER=your-mqtt-broker.com
export INFLUXDB_HOST=your-influxdb.com
export INFLUXDB_DB=iot_tanks_prod
```

### 2. PM2 Process Manager
```bash
npm install -g pm2
pm2 start node-red --name "iot-dashboard"
pm2 save
pm2 startup
```

### 3. Reverse Proxy
Use Nginx or Apache as reverse proxy:
```nginx
server {
    listen 80;
    server_name your-domain.com;
    
    location / {
        proxy_pass http://localhost:1880;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

## Next Steps

1. **Customize the Flow**: Modify the Node-RED flow to match your specific sensor setup
2. **Add More Sensors**: Extend the flow to handle additional tank and flow meter sensors
3. **Implement Alerts**: Add alert generation based on sensor thresholds
4. **Data Analytics**: Add data processing nodes for advanced analytics
5. **User Management**: Implement user authentication and authorization
6. **Mobile App**: Consider creating a mobile app using the same API endpoints

## Support

For issues and questions:
1. Check Node-RED logs
2. Review this documentation
3. Check Node-RED community forums
4. Review sensor documentation 