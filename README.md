# HEDA Data Access API

The HEDA API provides station information, latest sensor readings and historical data. Obtain a token first, then use that token to query station data.

## Methods

| Method | Endpoint | Purpose |
|---|---|---|
| POST | [/hd/user/auth.json](#authentication) | Obtain an access token. |
| POST | [/hd/station/tree.json](#station-tree) | Discover station numbers and names from the station tree. |
| POST | [/hd/station/detaillist.json](#station-data) | Retrieve station details and sensor data for a time range. |

## Getting started
Ask HEDA for your customer ID (`Cid`), application ID (`Aid`), username and password. After authentication, use `/hd/station/tree.json` to discover station numbers and names for subsequent data queries. If querying a specific subtree, obtain its root node type and ID from HEDA.

Base URL for the documented deployment:

```text
http://175.138.67.155:7077
```

Send JSON with `Content-Type: application/json`. Preserve field spelling and capitalization.

1. Call `/hd/user/auth.json` with your credentials.
2. Check `Code` and `Success`, then copy `Response.Token`.
3. Call `/hd/station/tree.json` with the token. Traverse `Response.Children` recursively and select station nodes (`Type = "STATION"`).
4. Copy a selected station node's `StationNo` into `StationSns`, or its `Name` into `StationNms`. Both query parameters are arrays of strings.
5. Call `/hd/station/detaillist.json` with the same JSON-body `Token`, your selected station filter and the required time range.
6. Read `Response.Data[].Sensors[].Vals[]` for historical samples. Each sample contains `Time` and `Val`; the sensor's `Unit` supplies the measurement unit.

Examples contain placeholders and illustrative data, not live API captures. Replace all `YOUR_...` values before use. cURL examples use POSIX shell line continuations; other clients can use the same URL, header and JSON body.

## Common conventions

| Response field | JSON type | Meaning |
|---|---|---|
| `Code` | integer | Application result: `0` = success; non-zero = error. This is not the HTTP status code. |
| `Success` | boolean | `true` = success; `false` = failure. |
| `Message` | string | Result description; success examples use `OK`. |
| `Success` | boolean | Included in the authentication and detail examples: `true` = success; `false` = failure. Not included in the supplied tree response. |
| `Message` | string | Result description; authentication/detail examples use `OK`, while the tree example uses an empty string. |
| `Response` | object | Endpoint-specific success payload. Error payload structure is not specified. |

Process a response as successful when `Code` is `0` and `Success` is `true`. Treat disagreement as an unexpected response.
For authentication and detail queries, process a response as successful when `Code` is `0` and `Success` is `true`. For the tree query, check `Code = 0`; its supplied response does not include `Success`. If `Success` is present and contradicts `Code`, treat the response as unexpected.

`Begin`, `End` and token expiry `Exp` are documented as timestamps in **seconds**, not milliseconds. The original document does not explicitly specify the epoch or the units of every response time field. Confirm the Unix-seconds interpretation and response timestamp units with HEDA before production use. If Unix seconds are confirmed, convert UTC instants to seconds and apply a timezone only for display.

`Station.Time` is a display string (the original example is `09-24 14:00`). Its timezone and complete format are unspecified; do not use it to construct query boundaries.

<a id="authentication"></a>
## /hd/user/auth.json

### Purpose

Authenticate using your customer ID, application ID and credentials. Returns the token required for station-data requests.

### Signature

```text
POST http://175.138.67.155:7077/hd/user/auth.json
Content-Type: application/json
```

### Body

| Parameter | Required | JSON type | Description |
|---|---|---|---|
| `Cid` | Yes | string | Customer ID supplied by HEDA. |
| `Aid` | Yes | string | Application ID supplied by HEDA. Confirm the deployment value: the previous table says `Uniscada`, but its examples use `scada`. |
| `UserName` | Yes | string | Integration account username. |
| `Password` | Yes | string | Integration account password. |

### Return value

The common response envelope contains:

| Field | JSON type | Description |
|---|---|---|
| `Response.Token` | string | Access token for subsequent request bodies. |
| `Response.Aid` | string | Application ID of the authenticated session. |
| `Response.Cid` | string | Customer ID of the authenticated session. |
| `Response.Uid` | string | User ID. |
| `Response.UserName` | string | Returned user name; the source describes it as the user's real name. |
| `Response.Ext` | string or null | Extension field; the source example contains `null`. |
| `Response.Exp` | integer | Token-expiry timestamp in seconds. A fixed lifetime is not documented. |

### Example

Request:

```sh
curl --request POST 'http://175.138.67.155:7077/hd/user/auth.json' \
  --header 'Content-Type: application/json' \
  --data '{"Cid":"YOUR_CUSTOMER_ID","Aid":"YOUR_APPLICATION_ID","UserName":"YOUR_USERNAME","Password":"YOUR_PASSWORD"}'
```

Illustrative response:

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

Use `Response.Token`, not `Response.Uid`, as the credential. If the service reports an expired or invalid token, authenticate again.

<a id="station-tree"></a>
## /hd/station/tree.json

### Purpose

Query a hierarchical tree of divisions, stations and equipment nodes. Use the returned station nodes to obtain the filters needed by `station/detaillist.json`:

| Selected station-node field | Subsequent query parameter | Example |
|---|---|---|
| `StationNo` | `StationSns` | `"800"` → `["800"]` |
| `Name` | `StationNms` | `"No.1 Water Supply Station"` → `["No.1 Water Supply Station"]` |

The JSON field is **`Name`**, with an uppercase `N`, as shown in the supplied schema and example. `ObjId` identifies a tree node; do not substitute it for `StationNo` when constructing `StationSns`.

### Signature

```text
Endpoint: http://175.138.67.155:7077/hd/station/tree.json
Content-Type: application/json
```

The supplied specification includes a JSON request body but does not state the HTTP method. Confirm the method with HEDA before sending this request.

### Body

| Parameter | Required | JSON type | Description |
|---|---|---|---|
| `Token` | Yes | string | Access token returned by authentication. |
| `Type` | No | string | Type of the root node used as the query entry, for example `DIVISION`. |
| `ObjId` | No | string | Unique ID of the root entry node, for example `DIV_001`. |
| `EndType` | No | string | Terminal node-type filter, for example `STATION`. |

Default root behavior when `Type` or `ObjId` is omitted, supported type values and the exact pruning behavior of `EndType` are not specified. The following request illustrates a known division root with stations as the terminal type.

### Return value

`Response` is a root node object, not an array. Each node can contain a `Children` array of nodes with the same structure.

| Node field | JSON type | Description |
|---|---|---|
| `Type` | string | Node classification; examples include `DIVISION` and `STATION`. |
| `Name` | string | Node display name. For a station node, use this in `StationNms`. |
| `StationNo` | string | Station number. For a station node, use this in `StationSns`. Availability on non-station nodes is unspecified. |
| `ObjId` | string | Unique node ID. |
| `Position` | object | Node center coordinates. |
| `Position.Lng` | number | Longitude, represented as a JSON number in this endpoint. |
| `Position.Lat` | number | Latitude, represented as a JSON number in this endpoint. |
| `Area` | array of objects | Coverage-area polygon coordinates; each entry contains numeric `Lng` and `Lat`. |
| `Children` | array of objects | Child nodes; traverse recursively to find stations beneath nested divisions. |

### Example

Request body (replace the illustrative root ID with a valid one for your account):

```json
{
  "Token": "YOUR_ACCESS_TOKEN",
  "Type": "DIVISION",
  "ObjId": "DIV_001",
  "EndType": "STATION"
}
```

Illustrative response. `StationNo` is included on the child station so the example can be used for the next query. `Area` retains the source's abbreviated coordinate example; polygon validation and closure rules are not specified.

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

### Use the station in a data query

From the example above:

- `Response.Children[0].StationNo` supplies `StationSns: ["800"]`.
- `Response.Children[0].Name` supplies `StationNms: ["No.1 Water Supply Station"]`.
- Use station nodes, not the division root's display name or ID. If divisions are nested, continue through their `Children` arrays.

Query by station number (recommended for the first request):

```json
{
  "Token": "YOUR_ACCESS_TOKEN",
  "StationSns": ["800"],
  "Begin": 1790697600,
  "End": 1790699400
}
```

Alternatively, replace `StationSns` with `"StationNms": ["No.1 Water Supply Station"]`. If you send both filters, the station must match both. Keep each number associated with its corresponding name when selecting stations.

Example JavaScript for collecting station choices from parsed tree JSON (`treeResult`):

```js
if (treeResult.Code !== 0 || treeResult.Success === false) {
  throw new Error(treeResult.Message || "HEDA station-tree request failed");
}
const stations = [];
function visit(node) {
  if (!node || typeof node !== "object") return;
  if (node.Type === "STATION") {
    stations.push({ StationNo: node.StationNo, Name: node.Name });
  }
  for (const child of Array.isArray(node.Children) ? node.Children : []) {
    visit(child);
  }
}
visit(treeResult.Response);
// Select the desired station(s) from this list before building a data request.
console.log(stations);
```

If a selected station lacks `StationNo`, do not use its `ObjId` as a replacement; use its valid `Name` filter or confirm the station number with HEDA. Do not send an empty filter when no station was selected.

<a id="station-data"></a>
## /hd/station/detaillist.json

### Purpose

Retrieve station metadata, sensor metadata, latest readings, historical readings for a period, and alarm-related fields for matching stations.

For your first integration, query one station by number and read its sensor history. Latest readings and historical samples are separate parts of the response.

### Signature

```text
POST http://175.138.67.155:7077/hd/station/detaillist.json
Content-Type: application/json
```

### Body

| Parameter | Required | JSON type | Description |
|---|---|---|---|
| `Token` | Yes | string | `Response.Token` from authentication; send in this JSON body. |
| `StationSns` | No | array of strings | Exact station-number filter, for example `["800"]`. Use station numbers, not station object IDs. |
| `StationNms` | No | array of strings | Exact station-name filter. If both filters are supplied, the station must match both. |
| `StationSns` | No | array of strings | Exact station-number filter, for example `["800"]`. Obtain values from station-tree nodes' `StationNo`; do not use `ObjId`. |
| `StationNms` | No | array of strings | Exact station-name filter. Obtain values from station-tree nodes' `Name`. If both filters are supplied, the station must match both. |
| `Begin` | Yes | integer | Query start timestamp in seconds. |
| `End` | Yes | integer | Query end timestamp in seconds. |

For your first request, use only `StationSns`. Supplying both filters can exclude a station whose name differs from its number. Both filters are optional in the source, but omitted/empty-filter behavior is not documented; do not assume this lists all stations.
For your first request, use only `StationSns`. If supplying both filters, use the selected station's actual number and name; copying its number into the name filter can exclude the station. Both filters are optional in the source, but omitted/empty-filter behavior is not documented; do not assume this lists all stations.

Use a start time earlier than the end time. Maximum duration, boundary inclusivity, record limits, pagination and ordering are unspecified. 

### Return value

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
|---|---|---|
| `Response.Total` | integer | Number of matched stations; pagination semantics are unspecified. |
| `Response.Data` | array of objects | Station results. |
| `Response.Data[].Station` | object | Station metadata. |
| `Response.Data[].Sensors` | array of objects | Sensors, readings and alarm-related fields. |
| `Response.Data[].Divisions` | array of objects | Division metadata. |
| `Response.Data[].Group` | integer | Station group identifier. |
| `Response.Data[].TimeStamp` | integer | Latest station-data timestamp. |

Fields inside `Response.Data[].Station`:

| Field | JSON type | Description |
|---|---|---|
| `Id` | string | Unique station ID; different from the station number used in `StationSns`. |
| `Name` | string | Station name, used by `StationNms`. |
| `Sn` | string | Station number, used by `StationSns`. |
| `Ty` | integer | Station type code; enum values are unspecified. |
| `TyName` | string | Station type name. |
| `TimeStamp` | integer | Latest data timestamp. |
| `Time` | string | Display time; timezone and complete format are unspecified. |
| `Position` | object | Station coordinates. |
| `Position.Lat` | string | Latitude, represented as a string in the source. |
| `Position.Lng` | string | Longitude, represented as a string in the source. |

Fields inside `Response.Data[].Sensors[]`:

| Field | JSON type | Description |
|---|---|---|
| `Id` | string | Sensor identifier. |
| `ObjId` | string | Sensor object identifier. The example matches `Id`; equality is not guaranteed. |
| `Name` | string | Sensor name. |
| `Unit` | string | Measurement unit. `m³` describes volume, not a flow rate. |
| `Dp` | integer | Number of decimal places. |
| `Time` | integer | Timestamp of the latest reading. |
| `Value` | number | Latest sensor reading. |
| `DType` | string | Sensor data-type code; the source example is `SJLJ`. Full enum unspecified. |
| `Vals` | array of objects | Historical readings for the query period. |
| `Vals[].Time` | integer | Historical sample timestamp. |
| `Vals[].Val` | number | Historical sample value. Note `Val`, not `Value`. |

### Example

Request station `800` for a 1,800-second period. Replace the station number with one available to your account. If Unix seconds are confirmed, this interval is 2026-09-29 16:00–16:30 UTC.

```sh
curl --request POST 'http://175.138.67.155:7077/hd/station/detaillist.json' \
  --header 'Content-Type: application/json' \
  --data '{"Token":"YOUR_ACCESS_TOKEN","StationSns":["800"],"Begin":1790697600,"End":1790699400}'
```

Illustrative response showing core reading fields. Other fields are intentionally omitted from this example; the API has not been changed to remove them.

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
          "Name": "800",
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

How to read this result:

- `Response.Total = 1`: one station matches the filter.
- `Station.Sn = "800"`: identifies the station.
- `Sensors[0].Name = "Net Totalizer"`, with `Unit = "m³"`: identifies the measurement and unit.
- `Sensors[0].Value = 851.877`: latest reading, associated with `Sensors[0].Time`.
- `Sensors[0].Vals`: two historical samples, both strictly inside the query interval. This avoids assuming inclusive boundaries.

A latest reading is not a substitute for history. Do not assume it is the last sample of any requested historical period. Iterate both arrays for multiple stations and sensors.

Example JavaScript parsing logic, after receiving parsed JSON as `result`:

```js
if (result.Code !== 0 || result.Success !== true) {
  throw new Error(result.Message || "HEDA request failed");
}
for (const item of result.Response.Data) {
  for (const sensor of item.Sensors) {
    for (const sample of sensor.Vals) {
      console.log(item.Station.Sn, sensor.Id, sample.Time, sample.Val, sensor.Unit);
    }
  }
}
```

This illustrates the documented success shape. Confirm missing-field and null behavior before building production parsing logic. Do not convert missing/null readings into zero.

## Alarm fields and additional metadata

These fields are secondary to the basic history integration. This rewrite does not remove them from the API.

Fields inside `Response.Data[].Sensors[]`:

| Field | JSON type | Description |
|---|---|---|
| `AlarmType` | string | Alarm type name. |
| `Alarm_r` | integer | Recovery flag: `0` = not recovered; `1` = recovered. |
| `STime` | integer | Alarm start timestamp. |
| `Ref` | number | Alarm threshold reference value. |
| `Level` | integer | Alarm level; severity mapping unspecified. |
| `Confirmed` | integer | `0` = unconfirmed; `1` = confirmed. |
| `Dispatch` | integer | Work-order dispatch flag: `0` = no; `1` = yes. |

The source does not define how to determine whether an alarm currently exists. Do not treat `Alarm_r = 0` alone as proof of an active alarm.

The original example includes the following additional fields without sufficient definitions. Presence in a sample does not establish a required field, default, complete type or stable enum. Request definitions if your integration needs them.

| Object | Additional fields observed in the original example |
|---|---|
| `Divisions[]` | `dt`, `Id`, `Weight`, `Name` |
| `Sensors[]` | `SType`, `MN`, `MX`, `Weight`, `Type`, `Alarm_t`, `Alarm_v`, `ConfirmInfo`, `Gdbh`, `Title`, `isShowDL`, `bgbj_id`, `Group`, `GroupNm`, `extend_sns`, `Lastycolor`, `Befcolor`, `dltime`, `Lastmcolor`, `At`, `Count`, `Curcolor`, `VType`, `Yescolor`, `Gd` |
| `Sensors[].Vals[]` | `qval`, `yval`, `lval`, `Report`, `mval` |
| `Station` | `USN`, `NoiseSbbh`, `Fav`, `Weight`, `KWeight`, `Diam`, `No`, `AZWZ`, `PId`, `Sjgs`, `Tl`, `Zoom`, `GID`, `Dp`, `DpName`, `Pic`, `State`, `Glzd`, `Im` |

## Troubleshooting

| Symptom | What to check |
|---|---|
| Authentication fails | Confirm `Cid`, deployment-specific `Aid`, username and password. |
| Non-zero `Code` or `Success = false` | Inspect `Message`; check fields, permissions and token validity. |
| Invalid or expired token | Authenticate again and replace the JSON-body `Token`. |
| Station not returned | Check station number, exact spelling and account permissions. If both filters are supplied, check their intersection. |
| Missing/unexpected history | Check timestamp units, interval and sensor `Vals`; a latest reading does not guarantee history for the interval. |
| HTTP/connection error or non-JSON response | Check deployment URL and HTTP response before parsing the application result. |

Numeric error codes, HTTP error mappings and example error bodies are unspecified in the source and are not invented here.

## Deployment details to confirm

- Correct `Aid`: the previous document conflicts between `Uniscada` and `scada`.
- Timestamp epoch, units of all response time fields, and display timezone.
- Query boundaries, maximum interval, result limits, pagination and ordering.
- Omitted/empty filters, no-match/no-history responses and nullable fields.
- Token lifetime, error-code definitions and station discovery if required.
- Token lifetime and error-code definitions.
- Tree-query HTTP method, default root selection and supported node types.
- Approved deployment URL and HTTPS availability; the source specifies HTTP only.
