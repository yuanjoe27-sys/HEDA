# HEDA Data Access API

The HEDA API provides station information, latest sensor readings, historical data and alarm-related fields. Authenticate first, then send the returned token in station-data request bodies.

## Contents

- [API overview](#api-overview)
- [Authentication](#authentication)
- [Station tree](#station-tree)
- [Station details and sensor data](#station-details-and-sensor-data)

## API overview

| Method | Endpoint | Purpose |
| --- | --- | --- |
| POST | `/hd/user/auth.json` | Obtain an access token. |
| POST | `/hd/station/tree.json` | Discover station numbers and names. |
| POST | `/hd/station/detaillist.json` | Retrieve station details and sensor data for a time range. |

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

## Station tree

Endpoint: `https://heda.cosmos-ins.com.my:7077/hd/station/tree.json`  
Content type: `application/json`  

### Request parameters

| Parameter | Required | JSON type | Description |
| --- | --- | --- | --- |
| `Token` | Yes | string | Access token returned by authentication. |

### Response fields

`Response` is a root node object. Its `Children` array contains nodes with the same structure.

| Node field | JSON type | Description |
| --- | --- | --- |
| `Type` | string | Node classification; examples include `DIVISION` and `STATION`. |
| **`Name`** | string | **Key field on station nodes: save for the next request's `StationNms` array.** Names on division nodes are not station names. |
| **`StationNo`** | string | **Key field on station nodes: save for the next request's `StationSns` array.** Availability on other node types is unspecified. |
| `ObjId` | string | Unique tree-node ID. |
| `Position` | object | Node center coordinates. |
| `Position.Lng` | number | Longitude. |
| `Position.Lat` | number | Latitude. |
| `Area` | array of objects | Coverage-area coordinates, each containing numeric `Lng` and `Lat`. |
| `Children` | array of objects | Child nodes. Traverse recursively to find nested stations. |

### Example request body

```json
{
  "Token": "YOUR_ACCESS_TOKEN"
}
```

### Example response

The `Area` example is abbreviated; it does not establish polygon validation or closure rules.

Comments below are for documentation only; the actual API response uses standard JSON.

```jsonc
{
  "Code": 0, 
  "Response": { 
    "Type": "DIVISION", // Can be ignored.
    "Name": "East Operation Zone", // Can be ignored.
    "ObjId": "DIV_001", // Can be ignored.
    "Position": {"Lng": 120.123456, "Lat": 30.654321}, // Can be ignored.
    "Area": [ // Can be ignored.
      {"Lng": 120.123, "Lat": 30.654},
      {"Lng": 120.125, "Lat": 30.656}
    ],
    "Children": [ 
      {
        "Type": "STATION", // Can be ignored.
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

### Map a station to a data-query filter

| Station-node field | Query parameter | Example |
| --- | --- | --- |
| `StationNo` | `StationSns` | `"111"` → `["111"]` |
| `Name` | `StationNms` | `"PJ.821 USJ 1 Subang Permai (MP1)"` → `["PJ.821 USJ 1 Subang Permai (MP1)"]` |

Select station nodes rather than the division root. Use `StationNo` rather than `ObjId` for `StationSns`.

For the station in the example, the next data-query request can use:

```json
{
  "Token": "YOUR_ACCESS_TOKEN",
  "StationSns": ["111"],
  "StationNms": 'PJ.821 USJ 1 Subang Permai (MP1)', 
  "Begin": 1790697600,
  "End": 1790699400
}
```

Replace the token, station number and time range with your actual values. To filter by name instead, replace `StationSns` with `"StationNms": ["PJ.821 USJ 1 Subang Permai (MP1)"]`. Either filter can be used; both are not required. If both are supplied, the station must match both filters.

## Station details and sensor data

Retrieve station metadata, sensor metadata, latest readings, historical samples and alarm-related fields for matching stations.

```https
POST https://heda.cosmos-ins.com.my:7077/hd/station/detaillist.json
Content-Type: application/json
```

### Request parameters

| Parameter | Required | JSON type | Description |
| --- | --- | --- | --- |
| `Token` | Yes | string | `Response.Token` returned by authentication. |
| `StationSns` | No | array of strings | Exact station numbers from tree nodes' `StationNo`, for example `["111"]`. |
| `StationNms` | No | array of strings | Exact station names from tree nodes' `Name`. |
| `Begin` | Yes | integer | Query start timestamp in seconds. |
| `End` | Yes | integer | Query end timestamp in seconds. |

Use a start time earlier than the end time. If both station filters are supplied, a station must match both; use its actual number and name. Behavior for omitted or empty filters is unspecified.

### Response structure

```text
Response
├── Total                         Number of matched stations
└── Data[]                        Station results
    ├── Station                   Station identity and location
    ├── Sensors[]                 Sensors belonging to this station
    │   ├── Unit                  Measurement unit
    │   ├── Time / Value          Latest reading
    │   └── Vals[]                Historical samples
    │       └── Time / Val        Sample timestamp and value
    ├── Divisions[]               Division metadata
    ├── Group                     Station group identifier
    └── TimeStamp                 Latest station-data timestamp
```

| Field path | JSON type | Description |
| --- | --- | --- |
| `Response.Total` | integer | Number of matched stations. |
| `Response.Data` | array of objects | Station results. |
| `Response.Data[].Station` | object | Station metadata. |
| `Response.Data[].Sensors` | array of objects | Sensors, readings and alarm-related fields. |
| `Response.Data[].Divisions` | array of objects | Division metadata. |
| `Response.Data[].Group` | integer | Station group identifier. |
| `Response.Data[].TimeStamp` | integer | Latest station-data timestamp. |

### Station fields

Fields inside `Response.Data[].Station`:

| Field | JSON type | Description |
| --- | --- | --- |
| `Id` | string | Unique station ID. |
| `Name` | string | Station name, used by `StationNms`. |
| `Sn` | string | Station number, used by `StationSns`; corresponds to tree-node `StationNo`. |
| `Ty` | integer | Station type code; enum values are unspecified. |
| `TyName` | string | Station type name. |
| `TimeStamp` | integer | Latest data timestamp. |
| `Time` | string | Display time; timezone and complete format are unspecified. |
| `Position` | object | Station coordinates. |
| `Position.Lat` | string | Latitude, represented as a string in this endpoint. |
| `Position.Lng` | string | Longitude, represented as a string in this endpoint. |

### Sensor fields

Fields inside `Response.Data[].Sensors[]`:

| Field | JSON type | Description |
| --- | --- | --- |
| `Id` | string | Sensor identifier. |
| `ObjId` | string | Sensor object identifier. The example matches `Id`; equality is not guaranteed. |
| `Name` | string | Sensor name. |
| `Unit` | string | Measurement unit. `m³` represents volume. |
| `Dp` | integer | Number of decimal places. |
| `Time` | integer | Timestamp of the latest reading. |
| `Value` | number | Latest sensor reading. |
| `DType` | string | Sensor data-type code; the example uses `SJLJ`. Full enum is unspecified. |
| `Vals` | array of objects | Historical readings for the query period. |
| `Vals[].Time` | integer | Historical sample timestamp. |
| `Vals[].Val` | number | Historical sample value. The field name differs from the latest reading's `Value`. |

### Example request

Query station `111` for a 1,800-second interval. Replace the number with one available to your account.

```bash
curl --request POST 'https://heda.cosmos-ins.com.my:7077/hd/station/detaillist.json' \
  --header 'Content-Type: application/json' \
  --data '{"Token":"YOUR_ACCESS_TOKEN","StationSns":["111"],"Begin":1790697600,"End":1790699400}'
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
      {
        "Station": {
          "Id": "EXAMPLE_STATION_ID",
          "Name": "PJ.821 USJ 1 Subang Permai (MP1)",
          "Sn": "111",
          "Ty": 808,
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

The result contains one station (`Sn = "111"`). Its sensor's latest value is `851.877 m³`, associated with `Sensors[0].Time`. `Vals` contains two historical samples strictly inside the requested interval; this example does not establish boundary inclusivity.
