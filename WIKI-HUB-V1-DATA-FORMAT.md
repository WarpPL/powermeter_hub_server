# Hub v1 Data Format

## Overview

Hub v1 uses a different communication protocol and data format compared to v2/v3 hubs. It sends JSON-encoded data with milliamp readings that must be converted to watts.

---

## Network Communication

### Domain Pattern
- **Data submission**: `[MAC].sensornet.info`
- **Key checking**: `[MAC].keys.sensornet.info`

**Example:**
- `aa.bb.cc.dddddd.sensornet.info`
- `aa.bb.cc.dddddd.keys.sensornet.info`

*Note: v2/v3 hubs use `[MAC].[h2/h3].sensornet.info` pattern*

### User Agent
```
Energyhive Hub/v1.0.1
```

---

## Endpoints

### 1. Data Submission: `POST /recjson`

**Content-Type:** `application/x-www-form-urlencoded`

**Request Body Format:**
```
json=[MAC]|[TIMESTAMP]|[VERSION]|[JSON_DATA]|[HASH]
```

**Example:**
```
json=AABBCCDDDDDD|694851F9|v1.0.1|{"data":[[610965,"mA","E1",33314,0,0,65535]]}|39ef0bdc14b52df375b79555f059b52f
```

**Field Breakdown:**

| Position | Field            | Example                            | Description                              |
|----------|------------------|------------------------------------|------------------------------------------|
| 0        | MAC Address      | `AABBCCDDDDDD`                     | Hub's MAC address (reversed from domain) |
| 1        | Timestamp        | `694851F9`                         | Hexadecimal timestamp                    |
| 2        | Firmware Version | `v1.0.1`                           | Hub firmware version                     |
| 3        | JSON Data        | `{"data":[[...]]}`                 | Sensor readings in JSON format           |
| 4        | Hash             | `39ef0bdc14b52df375b79555f059b52f` | MD5 hash for verification                |

**JSON Data Structure:**
```json
{
  "data": [
    [
      610965,      // Position 0: Unknown (sensor ID or timestamp?)
      "mA",        // Position 1: Unit (milliamps)
      "E1",        // Position 2: Sensor port identifier
      33314,       // Position 3: Current reading in milliamps
      0,           // Position 4: Unknown
      0,           // Position 5: Unknown
      65535        // Position 6: Unknown (possibly max value or status?)
    ]
  ]
}
```

**Key Field:**
- **Position 3**: Current reading in **milliamps** (mA)

**Expected Response:**
```
success
```

**Frequency:** Every ~6 seconds (but depends on clamp sensor setting)

---

### 2. Key Checking: `GET /check_key.html`

**Query Parameters:**
- `p` - Sensor port (e.g., `E1`)
- `ts` - Timestamp (e.g., `00000000`)
- `h` - Hash (e.g., `e660718d35a6adb36ca2a6440035af02`)

**Example Request:**
```
GET /check_key.html?p=E1&ts=00000000&h=e660718d35a6adb36ca2a6440035af02
```

**Expected Response:**
```
success
```

**Purpose:** Authentication/validation check

**Frequency:** Periodic (less frequent than data submission)

---

## Data Conversion

### Milliamps to Watts Formula

```
watts = (MAINS_VOLTAGE × milliamps / 1000) × POWER_FACTOR
```

**Example Calculation:**
```
milliamps = 33314
MAINS_VOLTAGE = 230V (Europe) or 120V (North America)
POWER_FACTOR = 0.6 (typical) or 0.5 (adjusted for display matching)

watts = (230 × 33314 / 1000) × 0.6
watts = 7662.22 × 0.6
watts = 4597.332
```

**For North America (120V):**
```
watts = (120 × 33314 / 1000) × 0.5
watts = 3997.68 × 0.5
watts = 1998.84
```

### Sample Readings from Live Hub

| Timestamp | Milliamps | Calculated Watts (230V, 0.6 PF) |
|-----------|-----------|---------------------------------|
| 21:01:07  | 33314     | 4597.332                        |
| 21:01:11  | 27959     | 3858.342                        |
| 21:01:17  | 32099     | 4429.662                        |
| 21:01:36  | 28724     | 3963.912                        |
| 21:01:41  | 30117     | 4156.146                        |
| 21:01:47  | 30431     | 4199.478                        |
| 21:01:53  | 27548     | 3801.624                        |
| 21:01:59  | 28019     | 3866.622                        |
| 21:02:05  | 32250     | 4450.5                          |

---

## Database Storage

### Label Format
```
efergy_h1_[VERSION]_[MAC]
```

**Example:**
```
efergy_h1_v1.0.1_AABBCCDDDDDD
```

### Stored Value
**Watts** (already converted from milliamps)

**Example Database Entry:**
```
label: efergy_h1_v1.0.1_AABBCCDDDDDD
timestamp: 1734814867
value: 4597.332  # Watts
```

---

## Energy Aggregation

During hourly aggregation, v1 values are treated as watts and simply converted to kW:

```sql
WHEN labels.label LIKE 'efergy_h1%%'
    THEN readings.value / 1000.0  -- Watts to kW
```

**Energy calculation:**
```
kWh = (watts / 1000) × (time_interval_seconds / 3600)
```

---

## Key Differences from v2/v3

| Aspect | Hub v1 | Hub v2/v3 |
|--------|--------|-----------|
| **Endpoint** | `/recjson` | `/h2` or `/h3` |
| **Content-Type** | `application/x-www-form-urlencoded` | Raw body |
| **Data Format** | `json=` prefixed, URL-encoded | Pipe-delimited, raw |
| **Unit** | Milliamps (JSON) | Raw sensor values |
| **Conversion** | At logging time | During aggregation |
| **Domain Pattern** | `[MAC].sensornet.info` | `[MAC].[h2/h3].sensornet.info` |
| **Response** | `success` | `\n` (newline) |
| **Data Structure** | JSON array | Pipe-delimited string |
