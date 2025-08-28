# Architecture Decision: Raw Sensor Data vs Derived Calculations

## **Decision: Node-RED Sends Raw Sensor Data, Dashboard Derives Calculated Values** 🎯

### **Overview**
This document explains why we chose to have Node-RED send only raw sensor data to the dashboard, while the dashboard service handles all derived calculations and business logic.

## **Architecture Diagram**

```
IoT Sensors → MQTT → Node-RED → Raw Data Storage → API/WebSocket → Dashboard → Derived Values
                                    ↓
                              InfluxDB (Raw)
                                    ↓
                              Dashboard Service
                                    ↓
                              Business Logic
                                    ↓
                              User Interface
```

## **Why This Architecture?**

### **1. Single Source of Truth** 🎯
- **Raw sensor data** becomes the authoritative source
- **Derived calculations** are consistent across all consumers
- Eliminates risk of calculation discrepancies between systems

### **2. Separation of Concerns** 🏗️
- **Node-RED**: Data ingestion, validation, routing, storage
- **Dashboard**: Data presentation, business logic, user interface
- **Database**: Raw data storage, historical analysis

### **3. Scalability & Flexibility** 📈
- Multiple dashboards can derive different metrics from same raw data
- Business logic changes don't require Node-RED redeployment
- Easier to add new derived calculations

### **4. Data Integrity** 🔒
- Raw sensor values are preserved exactly as received
- Derived calculations can be recalculated if business rules change
- Historical data remains valid even if calculation logic evolves

### **5. Testing & Debugging** 🧪
- Raw data can be replayed to test different calculation scenarios
- Easier to debug calculation issues
- Can validate sensor calibration independently

## **Data Flow**

### **1. Sensor Data Ingestion**
```json
// MQTT Topic: sensors/tank1
{
  "levelRaw": 512,        // Raw ADC value (0-1023)
  "temperatureRaw": 245,   // Raw temperature sensor value
  "pressureRaw": 128,      // Raw pressure sensor value
  "flowRateRaw": 45,       // Raw flow sensor value
  "sensorId": "TANK001",
  "sensorType": "ultrasonic",
  "unit": "ADC"
}
```

### **2. Node-RED Processing**
- Validates data format
- Adds timestamp
- Stores raw data in InfluxDB
- Broadcasts via WebSocket

### **3. Dashboard Processing**
- Receives raw sensor data
- Applies calibration factors
- Calculates derived values
- Implements business logic
- Updates UI

### **4. Derived Values Examples**
```typescript
// Raw sensor data
levelRaw: 512

// Calibration factors
minRaw: 0, maxRaw: 1023
minLevel: 0%, maxLevel: 100%

// Derived calculation
level = ((512 - 0) / (1023 - 0)) * (100 - 0) + 0 = 50%

// Business logic
status = level < 15 ? 'critical' : level < 30 ? 'low' : 'normal'
volumeRemaining = (50 / 100) * 5000 = 2500L
```

## **Benefits of This Approach**

### **✅ Advantages**

1. **Data Accuracy**
   - Raw sensor values are never modified
   - Calibration can be adjusted without data loss
   - Historical data remains valid

2. **Flexibility**
   - Easy to change business rules
   - Can add new derived metrics
   - Supports multiple dashboard views

3. **Maintainability**
   - Clear separation of responsibilities
   - Easier to debug issues
   - Simpler testing

4. **Scalability**
   - Multiple consumers can process same raw data
   - Easy to add new sensors
   - Supports different calculation requirements

5. **Reliability**
   - Raw data backup ensures no information loss
   - Can reprocess data if needed
   - Fault tolerance in calculation layer

### **⚠️ Considerations**

1. **Processing Overhead**
   - Dashboard must process raw data
   - Slightly more complex client-side logic
   - Need for efficient calculation algorithms

2. **Configuration Management**
   - Calibration factors must be managed
   - Business rules need configuration
   - More complex setup initially

3. **Data Volume**
   - Raw data might be larger
   - Need efficient data transmission
   - Storage considerations

## **Implementation Details**

### **1. Node-RED Configuration**
```javascript
// Tank sensor processor
const msg = {
  payload: {
    id: 'tank-1',
    sensorId: msg.payload.sensorId || 'TANK001',
    timestamp: new Date().toISOString(),
    // Raw sensor readings only
    levelRaw: msg.payload.levelRaw || 0,
    temperatureRaw: msg.payload.temperatureRaw || 0,
    pressureRaw: msg.payload.pressureRaw || 0,
    flowRateRaw: msg.payload.flowRateRaw || 0,
    // Sensor metadata
    sensorType: msg.payload.sensorType || 'ultrasonic',
    unit: msg.payload.unit || 'ADC'
  }
};
```

### **2. Dashboard Data Processing**
```typescript
// Data processing service
processTankData(rawData: TankSensorData): TankData {
  const config = this.getTankConfig(rawData.id);
  
  // Calculate derived values
  const level = this.calculateLevel(rawData.levelRaw, config.levelCalibration);
  const temperature = this.calculateTemperature(rawData.temperatureRaw, config.temperatureCalibration);
  const pressure = this.calculatePressure(rawData.pressureRaw, config.pressureCalibration);
  const status = this.calculateTankStatus(level, config.thresholds);
  
  return {
    ...rawData,
    level,
    temperature,
    pressure,
    status,
    // Additional derived metrics
    volumeRemaining: (level / 100) * config.capacity,
    estimatedEmptyTime: this.estimateEmptyTime(level, rawData.flowRateRaw, config.capacity)
  };
}
```

### **3. Configuration Management**
```typescript
interface TankConfig {
  id: string;
  name: string;
  capacity: number;
  liquidType: 'water' | 'oil' | 'chemical' | 'fuel';
  levelCalibration: {
    minRaw: number;
    maxRaw: number;
    minLevel: number;
    maxLevel: number;
    offset: number;
  };
  thresholds: {
    critical: number;
    low: number;
  };
}
```

## **Alternative Approaches Considered**

### **❌ Option 1: Node-RED Calculates Everything**
- **Pros**: Simpler dashboard, less client-side processing
- **Cons**: Business logic in Node-RED, harder to maintain, less flexible

### **❌ Option 2: Hybrid Approach**
- **Pros**: Some calculations in Node-RED, some in dashboard
- **Cons**: Unclear responsibilities, potential inconsistencies, harder to debug

### **✅ Option 3: Raw Data + Dashboard Processing (Chosen)**
- **Pros**: Clear separation, flexible, maintainable, scalable
- **Cons**: More complex initial setup, requires configuration management

## **Performance Considerations**

### **1. Calculation Efficiency**
- Use efficient algorithms for real-time calculations
- Cache frequently used values
- Implement lazy loading for historical data

### **2. Data Transmission**
- Compress raw data if needed
- Use efficient WebSocket protocols
- Implement data batching for multiple sensors

### **3. Storage Optimization**
- Raw data in time-series database (InfluxDB)
- Derived values cached in memory
- Implement data retention policies

## **Security & Validation**

### **1. Data Validation**
- Validate raw sensor data format
- Check for reasonable value ranges
- Implement sensor health monitoring

### **2. Access Control**
- Secure API endpoints
- Authenticate WebSocket connections
- Implement role-based access control

### **3. Data Integrity**
- Checksums for critical data
- Audit logging for changes
- Backup and recovery procedures

## **Monitoring & Maintenance**

### **1. Health Checks**
- Monitor sensor data quality
- Track calculation performance
- Alert on anomalies

### **2. Configuration Management**
- Version control for configurations
- Automated deployment
- Rollback procedures

### **3. Performance Metrics**
- Response time monitoring
- Throughput measurements
- Resource utilization tracking

## **Future Enhancements**

### **1. Machine Learning Integration**
- Predictive maintenance
- Anomaly detection
- Optimization algorithms

### **2. Advanced Analytics**
- Trend analysis
- Pattern recognition
- Predictive modeling

### **3. Multi-tenant Support**
- User management
- Data isolation
- Custom dashboards

## **Conclusion**

The decision to separate raw sensor data from derived calculations provides a robust, scalable, and maintainable architecture for IoT tank monitoring. While it requires more initial setup and configuration management, the long-term benefits in terms of flexibility, reliability, and scalability far outweigh the initial complexity.

This architecture supports:
- **Real-time monitoring** with accurate data
- **Flexible business logic** that can evolve over time
- **Scalable deployment** for multiple sites and users
- **Reliable data storage** with full audit trail
- **Easy maintenance** and troubleshooting

The investment in proper architecture now will pay dividends as the system grows and requirements evolve. 