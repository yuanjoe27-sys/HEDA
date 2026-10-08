# HEDA Data Access API

The HEDA API provides station information, latest sensor readings, historical data and alarm-related fields. Authenticate first, then send the returned token in station-data request bodies.

## Contents

- [API overview](#api-overview)
- [Getting started](#getting-started)
- [Common conventions](#common-conventions)
- [Authentication](#authentication)
- [Station tree](#station-tree)
- [Station details and sensor data](#station-details-and-sensor-data)

## API overview

| Method | Endpoint | Purpose |
| --- | --- | --- |
| POST | `/hd/user/auth.json` | Obtain an access token. |
| POST | `/hd/station/tree.json` | Discover station numbers and names. |
| POST | `/hd/station/detaillist.json` | Retrieve station details and sensor data for a time range. |

The source lists POST for the station-tree endpoint in its overview but says the method is unspecified in its endpoint description. Confirm the method with HEDA before using it.

## Getting started

Base URL for the documented deployment:

```text
http://175.138.67.155:7077
```

Obtain your customer ID (`Cid`), application ID (`Aid`), username and password from HEDA. To query a specific subtree, also obtain its root node type and ID.

1. Call `/hd/user/auth.json` with your credentials.
2. Check the result and copy `Response.Token`.
3. Call `/hd/station/tree.json`. Inspect the root and traverse `Children` recursively to find nodes with `Type = "STATION"`.
4. Use a station's `StationNo` in `StationSns`, or its `Name` in `StationNms`. Both filters are arrays of strings.
5. Call `/hd/station/detaillist.json` with the token, station filter and time range. For the first request, use one station number.
6. Read historical samples from `Response.Data[].Sensors[].Vals[]`. Each sample has `Time` and `Val`; the sensor's `Unit` supplies the measurement unit.

## Common conventions

- Send JSON with `Content-Type: application/json`.
- Preserve field spelling and capitalization.
- Send `Token` in the JSON request body of station endpoints.
- Examples use placeholders and illustrative data. Replace `YOUR_...` values and example station/root identifiers before use.
- cURL examples use POSIX shell line continuations.

| Response field | JSON type | Meaning |
| --- | --- | --- |
| `Code` | integer | Application result: `0` = success; non-zero = error. This is separate from the HTTP status code. |
| `Success` | boolean | `true` = success; `false` = failure. Present in the authentication and detail examples; absent from the supplied tree example. |
| `Message` | string | Result description. Authentication/detail examples use `OK`; the tree example uses an empty string. |
| `Response` | object | Endpoint-specific payload. Error payload structure is unspecified. |

Check `Code` and, when present, `Success` before reading the payload. If the service reports an invalid or expired token, authenticate again.

Timestamps described below are in seconds. Their epoch and timezone interpretation still require confirmation; do not infer a fixed token lifetime from the examples.

## Authentication

Authenticate with your customer ID, application ID and account credentials to obtain a token.

```http
POST /hd/user/auth.json
Content-Type: application/json
```

### Request parameters

| Parameter | Required | JSON type | Description |
| --- | --- | --- | --- |
| `Cid` | Yes | string | Customer ID supplied by HEDA. |
| `Aid` | Yes | string | Application ID supplied by HEDA. Use the value for your deployment. |
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
curl --request POST 'http://175.138.67.155:7077/hd/user/auth.json' \
  --header 'Content-Type: application/json' \
  --data '{"Cid":"YOUR_CUSTOMER_ID","Aid":"YOUR_APPLICATION_ID","UserName":"YOUR_USERNAME","Password":"YOUR_PASSWORD"}'
```

### Example response

```json
{
  "Code": 0,
  "Success": true,
  "Message": "OK",
  "Response": {
    "Token": "YOUR_ACCESS_TOKEN",
    "Aid": "YOUR_APPLICATION_ID",
    "Cid": "YOUR_CUSTOMER_ID",
    "Uid": "EXAMPLE_USER_ID",
    "UserName": "YOUR_USERNAME",
    "Ext": null,
    "Exp": 1790704800
  }
}
```

## Station tree

Query the hierarchy of divisions, stations and equipment nodes. Station nodes provide the filters for the station-data endpoint.

Endpoint: `/hd/station/tree.json`  
Content type: `application/json`  
HTTP method: **to be confirmed**.

### Request parameters

| Parameter | Required | JSON type | Description |
| --- | --- | --- | --- |
| `Token` | Yes | string | Access token returned by authentication. |
| `Type` | No | string | Root node type, for example `DIVISION`. |
| `ObjId` | No | string | Root node ID, for example `DIV_001`. |
| `EndType` | No | string | Terminal node-type filter, for example `STATION`. |

### Response fields

`Response` is a root node object. Its `Children` array contains nodes with the same structure.

| Node field | JSON type | Description |
| --- | --- | --- |
| `Type` | string | Node classification; examples include `DIVISION` and `STATION`. |
| `Name` | string | Display name; station nodes supply `StationNms`. |
| `StationNo` | string | Station number; station nodes supply `StationSns`. Availability on other node types is unspecified. |
| `ObjId` | string | Unique tree-node ID. |
| `Position` | object | Node center coordinates. |
| `Position.Lng` | number | Longitude. |
| `Position.Lat` | number | Latitude. |
| `Area` | array of objects | Coverage-area coordinates, each containing numeric `Lng` and `Lat`. |
| `Children` | array of objects | Child nodes. Traverse recursively to find nested stations. |

### Example request body

Replace `DIV_001` with a valid root ID for your account.

```json
{
  "Token": "YOUR_ACCESS_TOKEN",
  "Type": "DIVISION",
  "ObjId": "DIV_001",
  "EndType": "STATION"
}
```

### Example response

The `Area` example is abbreviated; it does not establish polygon validation or closure rules.

```json
{
  "Code": 0,
  "Response": {
    "Type": "DIVISION",
    "Name": "East Operation Zone",
    "ObjId": "DIV_001",
    "Position": {"Lng": 120.123456, "Lat": 30.654321},
    "Area": [
      {"Lng": 120.123, "Lat": 30.654},
      {"Lng": 120.125, "Lat": 30.656}
    ],
    "Children": [
      {
        "Type": "STATION",
        "Name": "No.1 Water Supply Station",
        "StationNo": "800",
        "ObjId": "STA_001",
        "Position": {"Lng": 120.124, "Lat": 30.655},
        "Area": [],
        "Children": []
      }
    ]
  },
  "Message": ""
}
```

### Map a station to a data-query filter

| Station-node field | Query parameter | Example |
| --- | --- | --- |
| `StationNo` | `StationSns` | `"800"` → `["800"]` |
| `Name` | `StationNms` | `"No.1 Water Supply Station"` → `["No.1 Water Supply Station"]` |

Select station nodes rather than the division root. Use `StationNo` rather than `ObjId` for `StationSns`.

## Station details and sensor data

Retrieve station metadata, sensor metadata, latest readings, historical samples and alarm-related fields for matching stations.

```http
POST /hd/station/detaillist.json
Content-Type: application/json
```

### Request parameters

| Parameter | Required | JSON type | Description |
| --- | --- | --- | --- |
| `Token` | Yes | string | `Response.Token` returned by authentication. |
| `StationSns` | No | array of strings | Exact station numbers from tree nodes' `StationNo`, for example `["800"]`. |
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

Query station `800` for a 1,800-second interval. Replace the number with one available to your account.

```bash
curl --request POST 'http://175.138.67.155:7077/hd/station/detaillist.json' \
  --header 'Content-Type: application/json' \
  --data '{"Token":"YOUR_ACCESS_TOKEN","StationSns":["800"],"Begin":1790697600,"End":1790699400}'
```

Alternatively, replace `StationSns` with `"StationNms": ["No.1 Water Supply Station"]`.

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
          "Name": "No.1 Water Supply Station",
          "Sn": "800",
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

The result contains one station (`Sn = "800"`). Its sensor's latest value is `851.877 m³`, associated with `Sensors[0].Time`. `Vals` contains two historical samples strictly inside the requested interval; this example does not establish boundary inclusivity.
