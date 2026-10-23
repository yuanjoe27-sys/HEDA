# HEDA Data Access API

The HEDA API provides site information, latest sensor readings, historical data and alarm-related fields. Authenticate first, then send the returned token in site-data request bodies.

## Contents

- [API overview](#api-overview)
- [Authentication](#authentication)
- [Site tree](#site-tree)
- [Site details and sensor data](#site-details-and-sensor-data)
- [Channel Parameter Definitions](#Channel-Parameter-Definitions)
## API overview

| Method | Endpoint | Purpose |
| --- | --- | --- |
| POST | `/hd/user/auth.json` | Returns authentication bearer token. |
| POST | `/hd/station/tree.json` | Returns site numbers and names. |
| POST | `/hd/station/detaillist.json` | Returns site details and sensor data for a time range. |

## Authentication

Authenticate with your customer ID, application ID and account credentials to obtain a token.

### Request parameters

| Parameter | Required | JSON type | Description |
| --- | --- | --- | --- |
| `Cid` | Yes | string | Customer ID supplied by HEDA. |
| `Aid` | Yes | string | Application ID supplied by HEDA. |
| `UserName` | Yes | string | Integration account username. |
| `Password` | Yes | string | Integration account password. |

### Response fields

| Field | JSON type | Description |
| --- | --- | --- |
| `Response.Token` | string | Access token for subsequent request bodies. |
| `Response.Aid` | string | Application ID of the authenticated session. |
| `Response.Cid` | string | Customer ID of the authenticated session. |
| `Response.Uid` | string | User ID. Use `Token` as the credential. |
| `Response.UserName` | string | Returned user name; the source describes it as the user's real name. |
| `Response.Ext` | string or null | Extension field; the example contains `null`. |
| `Response.Exp` | integer | Token-expiry timestamp in seconds. |

### Example request

```bash
POST endpoint https://heda.cosmos-ins.com.my:7077/hd/user/auth.json
Content-Type: application/json
Body:
{
  "Cid": "YOUR_CUSTOMER_ID",
  "Aid": "YOUR_APPLICATION_ID",
  "UserName": "YOUR_USERNAME",
  "Password": "YOUR_PASSWORD"
}
```

### Example response

**Key field: `Response.Token`. After confirming a successful response, save this token for subsequent API requests. Other user information fields can be ignored.**

```jsonc
{
  "Code": 0, // Required: check that the value is 0 (success).
  "Success": true, // Required: check that the value is true.
  "Message": "OK", // Only needed for troubleshooting when the request fails.
  "Response": {
    "Token": "YOUR_ACCESS_TOKEN", // REQUIRED: save this token for subsequent API requests.
    "Aid": "YOUR_APPLICATION_ID", // Can be ignored.
    "Cid": "YOUR_CUSTOMER_ID", // Can be ignored.
    "Uid": "EXAMPLE_USER_ID", // Can be ignored.
    "UserName": "YOUR_USERNAME", // Can be ignored.
    "Ext": null, // Can be ignored.
    "Exp": 1790704800 // Optional: token expiry timestamp (seconds).
  }
}
```

## Site tree

**Returns site numbers and names.**

### Request parameters

| Parameter | Required | JSON type | Description |
| --- | --- | --- | --- |
| `Token` | Yes | string | Access token returned by authentication. |

### Response fields

`Response` is a root node object. Its `Children` array contains nodes with the same structure.

| Node field | JSON type | Description |
| --- | --- | --- |
| `Type` | string | Node classification; examples include `DIVISION` and `site`. |
| **`Name`** | string | **Key field on site nodes: save for the next request's `StationNms` array.** Names on division nodes are not site names. |
| **`StationNo`** | string | **Key field on site nodes: save for the next request's `StationSns` array.** Availability on other node types is unspecified. |
| `ObjId` | string | Unique tree-node ID. |
| `Position` | object | Node center coordinates. |
| `Position.Lng` | number | Longitude. |
| `Position.Lat` | number | Latitude. |
| `Area` | array of objects | Coverage-area coordinates, each containing numeric `Lng` and `Lat`. |
| `Children` | array of objects | Child nodes. Traverse recursively to find nested sites. |

### Example request body

Endpoint: `https://heda.cosmos-ins.com.my:7077/hd/station/tree.json`  
Content type: `application/json`  

```json
{
  "Token": "YOUR_ACCESS_TOKEN"
}
```

### Example response

**Key field: `Response.Children.name` and `Response.Children.StationNo`. Other information fields can be ignored.**

```jsonc
{
  "Code": 0, 
  "Response": { 
    "Type": "D", // Can be ignored.
    "Name": "AIS test", // Can be ignored.
    "ObjId": "6aab4a964dce2c19843c639e", // Can be ignored.
    "Position": {"Lng": 120.123456, "Lat": 30.654321}, // Can be ignored.
    "Area": [ // Can be ignored.
      {"Lng": 120.123, "Lat": 30.654},
      {"Lng": 120.125, "Lat": 30.656}
    ],
    "Children": [ 
      {
        "Type": "site", // Can be ignored.
        "Name": "PJ.821 USJ 1 Subang Permai (MP1)", // KEY: save for StationNms in the next API request.
        "StationNo": "111", // KEY: save for StationSns in the next API request.
        "ObjId": "STA_001", // // Can be ignored.
        "Position": {"Lng": 120.124, "Lat": 30.655}, // Can be ignored.
        "Area": [], // Can be ignored.
        "Children": [] // Can be ignored.
      }
    ]
  },
  "Message": "" // Can be ignored.
}
```

## Site details and sensor data

Retrieve site metadata, sensor metadata, latest readings, historical samples and alarm-related fields for matching sites.

### Request parameters

| Parameter | Required | JSON type | Description |
| --- | --- | --- | --- |
| `Token` | Yes | string | `Response.Token` returned by authentication. |
| `StationSns` | No | array of strings | Exact site numbers from tree nodes' `StationNo`, for example `["111"]`. |
| `StationNms` | No | array of strings | Exact site names from tree nodes' `Name`. |
| `Begin` | Yes | integer | Query start timestamp in seconds. |
| `End` | Yes | integer | Query end timestamp in seconds. |

Use a start time earlier than the end time. If both site filters are supplied, a site must match both; use its actual number and name. Behavior for omitted or empty filters is unspecified.

### Response structure

```text
Response
├── Total                         Number of matched sites // Can be ignored.
└── Data[]                        site results
    ├── site                      site identity and location
    ├── Sensors[]                 Sensors belonging to this site 
    │   ├── Unit                  Measurement unit
    │   ├── Time / Value          Latest reading
    │   └── Vals[]                Historical samples
    │       └── Time / Val        Sample timestamp and value
    ├── Divisions[]               Division metadata    // Can be ignored.
    ├── Group                     site group identifier // Can be ignored.
    └── TimeStamp                 Latest site-data timestamp // Can be ignored.
```

| Field path | JSON type | Description |
| --- | --- | --- |
| `Response.Total` | integer | Number of matched sites. |
| `Response.Data` | array of objects | site results. |
| `Response.Data[].site` | object | site metadata. |
| **`Response.Data[].Sensors`** | array of objects | Sensors, readings and alarm-related fields. |
| `Response.Data[].Divisions` | array of objects | Division metadata. |
| `Response.Data[].Group` | integer | site group identifier. |
| `Response.Data[].TimeStamp` | integer | Latest site-data timestamp. |

### site fields

Fields inside `Response.Data[].site`:

| Field | JSON type | Description |
| --- | --- | --- |
| `Id` | string | Unique site ID. |
| `Name` | string | site name, used by `StationNms`. |
| `Sn` | string | site number, used by `StationSns`; corresponds to tree-node `StationNo`. |
| `Ty` | integer | site type code; enum values are unspecified. |
| `TyName` | string | site type name. |
| `TimeStamp` | integer | Latest data timestamp. |
| `Time` | string | Display time; timezone and complete format are unspecified. |
| `Position` | object | site coordinates. |
| `Position.Lat` | string | Latitude, represented as a string in this endpoint. |
| `Position.Lng` | string | Longitude, represented as a string in this endpoint. |

## Reading sensor historical values (`Vals`)

**Read historical sensor values from `Response.Data[].Sensors[].Vals[]`. Each entry contains `Val` (the historical value) and `Time` (its timestamp in seconds).**

1. Iterate through `Response.Data[]` to access each returned site.
2. Find the required sensor in `Sensors[]` using its `Id` or `Name`.
3. Iterate through that sensor's `Vals[]` array.
4. Read `Val` for the value and `Time` for the corresponding timestamp. Use the parent sensor's `Unit` as the measurement unit.

```text
Response.Data[]
└── Sensors[]
    ├── Name              Identify the required sensor
    ├── Unit              Measurement unit
    ├── Value             Latest reading
    └── Vals[]            Historical readings — read this array
        ├── Time          Sample timestamp (seconds)
        └── Val           Historical value — read this field
```

#### Example

```json
{
  "Name": "Net Totalizer",
  "Unit": "m³",
  "Value": 851.877,
  "Vals": [
    {"Time": 1790697900, "Val": 851.125},
    {"Time": 1790698800, "Val": 851.877}
  ]
}
```

The `Vals` array contains two historical readings:

| Time (seconds) | Val | Unit |
| --- | --- | --- |
| 1790697900 | 851.125 | m³ |
| 1790698800 | 851.877 | m³ |

> **Important:** Historical readings use `Vals[].Val`, while the latest reading uses `Value`. Field names are case-sensitive. If `Vals` is empty, it contains no historical samples; do not interpret an empty array as a value of zero.

### Example request

```bash
POST https://heda.cosmos-ins.com.my:7077/hd/station/detaillist.json
Content-Type: application/json

{
  "Token": "YOUR_ACCESS_TOKEN",
  "StationSns": ["102"],
  "StationNms": ["KROHNE FM"]`,
  "Begin": 1790697600, //Timestamps in seconds.
  "End": 1790699400  //Timestamps in seconds.
}
```

Alternatively, or replace `StationSns` with `"StationNms": ["PJ.821 USJ 1 Subang Permai (MP1)"]`. or both.

### Example response

This example shows core reading fields; additional metadata and alarm fields are omitted for readability.

```json
{
  "Code": 0,
  "Success": true,
  "Message": "OK",
  "Response": {
    "Total": 1,
    "Data": [
       "Divisions": [
          {
            "dt": "1",
            "Id": "6aab4a964dce2c19843c639e",
            "Weight": 14,
            "Name": "AIS test"
          }
        ],
      {
        "station": {
          "Id": "6a0126c8b7faa94c5c4ea57d",
          "Name": "KROHNE FM",
          "Sn": "102",
          "Ty": 1,
          "TyName": "HD86Q",
          "TimeStamp": 1790698800,
          "Position": {"Lat": "34.000000", "Lng": "113.000000"}
        },
        "Sensors": [
          {
            "Id": "PTAD_800_SJLJ",
            "ObjId": "PTAD_800_SJLJ",
            "Name": "Net Totalizer",
            "Unit": "m³",
            "Dp": 3,
            "Time": 1790698800,
            "Value": 851.877,
            "DType": "SJLJ",
            "Vals": [
              {"Time": 1790697900, "Val": 851.125},
              {"Time": 1790698800, "Val": 851.877}
            ]
          }
        ],
        "Group": 0,
        "TimeStamp": 1790698800
      }
    ]
  }
}
```
## Channel Parameter Definitions

Channel meanings depend on the device model. A dash (`—`) means no definition is provided for that model.

| Device | Channel | Siemens MAG8000 | KROHNE IFC070 | Unit | Remark |
| --- | --- | --- | --- | --- | --- |
| Meter | P1 | Forward totalizer | Forward totalizer | m³ | Cumulative forward volume. |
| Meter | P2 | Reverse totalizer | Reverse totalizer | m³ | Cumulative reverse volume. |
| Meter | SJLJ | Net totalizer | Net totalizer | m³ | Cumulative net volume. |
| Meter | A7 | Flow rate | Flow rate | m³/h | Instantaneous flow rate. |
| Meter | A8 | Velocity | — | mm/s | |
| Meter | A5 | Pressure_1 | — | mH₂O | Pressure channel 1; |
| Meter | A6 | Pressure_2 | — | mH₂O | Pressure channel 2; |
| Meter | A1 | Impedance A | Impedance A |  | Impedance measurement for channel A. |
| Meter | A2 | Impedance B | — |  | Impedance measurement for channel B. |
| Meter | A4 | Battery percentage | Battery Capacity | Battery percentage %; Battery Capacity Ah | Meter battery percentage; distinct from voltage and remaining capacity. |
| Meter | P7 | Diameter | — | mm | Pipe or meter diameter parameter. |
| Meter | P8 | Meter alarm code | Meter alarm code | — | See the [Meter Alarm Code Definitions](#meter-alarm-code-definitions) section below for alarm code descriptions. |
| Meter | Q1_12 | Serial_Com_Alm | Serial_Com_Alm | — | Communication status: 1 = normal; 2 = alarm. |
| Logger | V | Battery voltage | Battery voltage | V | Logger battery voltage. |
| Logger | S | Signal Quality (ASU) | Signal Quality (ASU) | — | Source thresholds: 0–84 = very good; 85–94 = strong; 95–104 = average; 105–114 = weak; >114 = very weak. |

### Meter Alarm Code Definitions

#### Siemens MAG8000

| Code | Description |
| ---: | --- |
| 1 | Insulation error |
| 2 | Coil current error |
| 4 | Preamplifier overload |
| 8 | Database checksum error |
| 16 | Low power warning |
| 32 | Flow overload warning |
| 64 | Pulse A overload warning |
| 128 | Pulse B overload warning |
| 256 | Consumption interval warning |
| 512 | Leakage warning |
| 1024 | Empty pipe warning |
| 2048 | Low impedance warning |
| 4096 | Flow limit warning |
| 8192 | Reverse flow warning |

#### KROHNE IFC070

| Code | Description |
| ---: | --- |
| 0 | No errors |
| 1 | Active flow measurement warning condition |
| 2 | Active battery below 10% warning condition |
| 4 | Active EEPROM error condition |
| 8 | Active communication error condition |
| 16 | Empty pipe |
| 32 | Mains power failure |
