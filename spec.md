# TransitPulse — Technical Specification

## 1. Simulated Telemetry JSON Schema (Sent every 10s per bus)
```json
{
  "bus_id": "BUS-014",
  "route_id": "R5",
  "gps_lat": 6.9271,
  "gps_lon": 79.8612,
  "speed_kmh": 22.4,
  "passenger_count": 47,
  "capacity": 50,
  "fuel_pct": 74.0,
  "timestamp": "2026-09-02T08:14:20Z",
  "seq": 88214,
  "device_signature": "sig:9f2a...c71"
}

## 2. In-Line Physical Validation Bounds
* Speed ceiling: 100 km/h. Calculations exceeding 120 km/h trigger spoof detection.
* Boarding rate limit: 1 passenger/second/door max.
* Geofence boundary: Active route corridor threshold is 500m.
* Device trust score: Valid range 0.00 to 1.00. Telemetry from devices under 0.40 is excluded from route planning.

## 3. Whitelisted System Action Commands
1. dispatch_relief
2. merge_trip
3. notify_passengers
4. escalate
5. no_action
