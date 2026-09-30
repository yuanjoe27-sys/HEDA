# HEDA Open API Specification Document

## Document Overview

This document is the open API documentation for the Station Data Query System. It covers two core interfaces: user authentication/login and station detail with historical data query. It standardizes the request URL, request parameters, response fields, and data samples for front-end and back-end development and integration debugging.

### General Conventions

- Protocol: HTTP
- Data Format: JSON
- Time Format: uniformly using
- Status Code Rule: `Code=0` indicates success; non-zero indicates an error

## 1. User Authentication API

### 1.1 Interface Basic Information

| Item | Content |
|---|---|
| Interface Name | User Authentication / Login |
| Request URL | `http://175.138.67.155:7077/hd/user/auth.json` |
| Request Method | `POST` |
| Description | Authenticate the user using Customer ID, Application ID, and account credentials to obtain a global access Token, which is required for all subsequent business interfaces. |

### 1.2 Request Parameters (Body JSON)

| Parameter | Required | Type | Description |
|---|---|---|---|
| `Cid` | Yes | String | Unique Customer ID |
| `Aid` | Yes | String | Unique Application ID, fixed value: `Uniscada` |
| `UserName` | Yes | String | Login username |
| `Password` | Yes | String | Login password |

### 1.3 Response Parameters

| Parameter | Type | Description |
|---|---|---|
| `Code` | Int | Status code: 0=success, non-zero=error |
| `Success` | Boolean | Request result: true=success, false=failure |
| `Message` | String | Result description; defaults to `"OK"` on success |
| `Response` | Object | Authentication data payload |
| `Response.Token` | String | Access credential; required for all subsequent business interfaces |
| `Response.Aid` | String | Application ID of the current authenticated session |
| `Response.Cid` | String | Customer ID of the current logged-in user |
| `Response.Uid` | String | Unique User ID |
| `Response.UserName` | String | Real name of the user |
| `Response.Ext` | String | Extension field; defaults to `null` |
| `Response.Exp` | Long | Token expiry timestamp (seconds) |

### 1.4 Request Example

```json
{
  "Cid": "673fdc7c421aa91379b266d8",
  "Aid": "scada",
  "UserName": "API",
  "Password": "Api2026!"
}
```

### 1.5 Response Example

```json
{
  "Code": 0,
  "Success": true,
  "Message": "OK",
  "Response": {
    "Token": "6a71b87ab7faa92fe0f3211a",
    "Aid": "scada",
    "Cid": "66de4c0c8c428f4730f6eb91",
    "Uid": "6a71b82bb7faa92fe0f3210f",
    "UserName": "API",
    "Ext": null,
    "Exp": 2736746291
  }
}
```

## 2. Station Detail and Data Query API

### 2.1 Interface Basic Information

| Item | Content |
|---|---|
| Interface Name | Station Detail and Time-range Data Query |
| Request URL | `http://175.138.67.155:7077/hd/station/detaillist.json` |
| Request Method | `POST` |
| Description | Authenticated via Token; queries the basic information, sensor information, real-time data, time-range historical data, and alarm information of the specified stations. |

### 2.2 Request Parameters (Body JSON)

| Parameter | Required | Type | Description |
|---|---|---|---|
| `Token` | Yes | String | Access credential obtained from the authentication interface |
| `StationSns` | No | Array[String] | Array of station numbers for exact-match query |
| `StationNms` | No | Array[String] | Array of station names for exact-match query; if both station numbers and names are provided, the intersection of results is returned |
| `Begin` | Yes | Long | Query start time (second-level timestamp) |
| `End` | Yes | Long | Query end time (second-level timestamp) |

### 2.3 Response Parameters

| Parameter | Type | Description |
|---|---|---|
| `Code` | Int | Status code: 0=success, non-zero=error |
| `Success` | Boolean | Request result: true=success, false=failure |
| `Message` | String | Result description |
| `Response` | Object | Data payload |
| `Response.Total` | Int | Total number of stations matched |
| `Response.Data` | Array[Object] | List of station data |
| `Data.Divisions` | Array[Object] | Station division information |
| `Data.Sensors` | Array[Object] | List of station sensors, their data, and alarm information |
| `Data.Station` | Object | Station basic information |
| `Data.TimeStamp` | Long | Timestamp of the station's latest data |
| `Data.Group` | Int | Station group identifier |

#### 2.3.1 Station Object - Field Description

| Parameter | Type | Description |
|---|---|---|
| `Id` | String | Unique station ID |
| `Name` | String | Station name |
| `Sn` | String | Station number |
| `Ty` | Int | Station type code |
| `TyName` | String | Station type name |
| `TimeStamp` | Long | Timestamp of the latest data |
| `Time` | String | Formatted time of the latest data |
| `Position` | Object | Station coordinate information |
| `Position.Lat` | String | Latitude |
| `Position.Lng` | String | Longitude |

#### 2.3.2 Sensors - Sensor and Alarm Data Field Description

| Parameter | Type | Description |
|---|---|---|
| `Id/ObjId` | String | Unique sensor tag/identifier |
| `Name` | String | Sensor name |
| `Unit` | String | Measurement unit |
| `Dp` | Int | Data precision (number of decimal places) |
| `Time` | Long | Timestamp of the latest data |
| `Value` | Float | Latest real-time value of the sensor |
| `DType` | String | Sensor data type code |
| `Vals` | Array[Object] | List of historical data within the queried time range |
| `Vals.Val` | Float | Data value at a time point |
| `Vals.Time` | Long | Timestamp corresponding to the data value |
| `AlarmType` | String | Alarm type name |
| `Alarm_r` | Int | Whether the alarm has recovered: 0=not recovered, 1=recovered |
| `STime` | Long | Alarm start timestamp |
| `Ref` | Float | Alarm threshold reference value |
| `Level` | Int | Alarm level |
| `Confirmed` | Int | Whether the alarm is confirmed: 0=not confirmed, 1=confirmed |
| `Dispatch` | Int | Whether the alarm has been dispatched as a work order: 0=no, 1=yes |

### 2.4 Request Example

```json
{
  "Token": "6a71b87ab7faa92fe0f3211a",
  "StationSns": ["800"],
  "StationNms": ["800"],
  "Begin": 1790697600,
  "End": 1790699400
}
```

### 2.5 Response Example

```json
{
  "Code": 0,
  "Success": true,
  "Message": "OK",
  "Response": {
    "Total": 1,
    "Data": [
      {
        "Divisions": [
          {
            "dt": "1",
            "Id": "66de5bb28c428f4730f6f2e2",
            "Weight": 1,
            "Name": "Testing"
          }
        ],
        "Sensors": [
          {
            "Id": "PTAD_800_SJLJ",
            "ObjId": "PTAD_800_SJLJ",
            "Name": "Net Totalizer",
            "SType": "66bc5bdf8c428f36ae784199",
            "Unit": "m³",
            "Dp": 3,
            "MN": null,
            "MX": null,
            "Time": 1790229600,
            "Value": 851.877,
            "Weight": 1,
            "DType": "SJLJ",
            "Vals": [
              {
                "qval": null,
                "Val": 17391.212,
                "yval": null,
                "lval": null,
                "Time": 1788192000,
                "Report": null,
                "mval": null
              }
            ],
            "Type": "hs",
            "AlarmType": "",
            "Alarm_r": 0,
            "STime": 1774800000,
            "Alarm_t": null,
            "Alarm_v": null,
            "Ref": 800,
            "Level": 3,
            "Confirmed": 1,
            "ConfirmInfo": null,
            "Dispatch": 0,
            "Gdbh": null,
            "Title": null,
            "isShowDL": 0,
            "bgbj_id": 0,
            "Group": "",
            "GroupNm": "[No Data]",
            "extend_sns": null,
            "Lastycolor": null,
            "Befcolor": null,
            "dltime": 0,
            "Lastmcolor": null,
            "At": "unknown",
            "Count": 5138,
            "Curcolor": null,
            "VType": "3",
            "Yescolor": null,
            "Gd": 0
          }
        ],
        "Station": {
          "Id": "689a95b9f766d221fe5f3429",
          "Name": "800",
          "Sn": "800",
          "Ty": 808,
          "TyName": "HD86Q",
          "USN": null,
          "NoiseSbbh": "",
          "Fav": false,
          "TimeStamp": 1790229600,
          "Time": "09-24 14:00",
          "Position": {
            "Lat": "34.7694704434",
            "Lng": "113.64590746"
          },
          "Weight": 2,
          "KWeight": null,
          "Diam": null,
          "No": "",
          "AZWZ": "",
          "PId": "",
          "Sjgs": null,
          "Tl": "",
          "Zoom": null,
          "GID": "",
          "Dp": null,
          "DpName": null,
          "Pic": 0,
          "State": null,
          "Glzd": [],
          "Im": 0
        },
        "Group": 0,
        "TimeStamp": 1790229600
      }
    ]
  }
}
```

## 3. Common Error Description

| Status / Scenario | Error Description | Resolution |
|---|---|---|
| Non-zero Code | Request error; `Message` returns the specific reason | Check the returned `Message` and validate the parameters, Token validity, and account permissions |
| Token Invalid | Token expired or invalid | Call the authentication interface again to obtain a new Token |
| Missing Parameters | Required parameter missing or malformed | Validate the completeness of request parameters, data types, and timestamp format |
