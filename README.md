# ShellyPro3EM Historian (Node-RED)

This is a Node-RED script which polls a remote ShellyPro3EM device via MQTT RPC (EMData.GetData / EMData.GetRecords) and writes the history to InfluxDB. 

The flow auto-primes (polls) the earliest available data when DB empty or when requests fall outside the device’s stored window. State is auto-on at deploy. Manual prime is supported (for debugging).

## Requirements

- Node-RED (tested on v4.0.9)
- InfluxDB **1.x** (tested on 1.8.x) – uses InfluxQL / database / measurement format  
  *(InfluxDB v2 is not supported unless compatibility mode is enabled)*  
- MQTT broker (Mosquitto, HiveMQ, EMQX, etc.)

The flow communicates directly with the Shelly device via MQTT RPC.

## Setup

1) Open the subflow and configure the MQTT broker and the InfluxDB connection nodes.

2) From the main flow, click the subflow instance to set the ENV vars below.

3) Deploy. The controller auto-enables state. Use 'manual prime' for debugging if needed (does affect off/on state).

### ENV vars

- **SHELLY_ID** :
Device ID (e.g. shellypro3em-XXXX). Used in tags and for "<deviceId>/rpc"

- **MQTT_PREFIX** :
Optional prefix before deviceId in the Shelly RPC topic. Empty = "<deviceId>/rpc"

- **MQTT_GETDATA_SRC** :
"src" string for GetData requests (e.g. emdata-getdata)

- **MQTT_GETDATA_TOPIC** :
MQTT topic for GetData responses (e.g. emdata-getdata/rpc)

- **MQTT_GETRECORDS_SRC** : 
"src" string for GetRecords requests (e.g. emdata-getrecords)

- **MQTT_GETRECORDS_TOPIC** : 
MQTT topic for GetRecords responses (e.g. emdata-getrecords/rpc)

- **INFLUX_DB** : 
InfluxDB database name (e.g. shellypro3em)

- **INFLUX_MEASUREMENT** :
Influx measurement (e.g. energy)

- **PRIME_THRESHOLD_MIN** :
Minutes since dbLastTs required before auto-prime (default 5, minimum 2)

## How it works

- **Control path 1 (CP1)** :
On start, queries Influx for the last timestamp. If none, triggers GetRecords (prime, aka CP2).
- **Data download loop** :
GetData → write points to Influx → advance dbLastTs → loop on interval.

- **Control path 2 (CP2)** : 
If GetData returns no data and no next_record_ts, and (now - dbLastTs) >= PRIME_THRESHOLD_MIN, trigger GetRecords.

- **Safety valve** :
After GetRecords, if dbLastTs >= device’s earliest ts (±120s), skip priming.

## Flow Control and debugging

- State toggle: send msg.state = true/false to the subflow input (default is true).
- Manual prime: send msg.action = "prime" (does not change state).
- For debugging, enable debug nodes on main flow.

## Context / diagnostics

- **history.dbLastIso**
ISO mirror of last DB timestamp written.

- **history.shellyLatestIso**
ISO mirror of device’s latest known timestamp.

- **history.shellyEarliestIso**
ISO mirror of device’s earliest known timestamp.

- **history.downloading**
Boolean flag while catching up.

- **history.lastPrimeInfo**
{ reason, shellyEarliestTs, dbLastTs, recordedAtTs, recordedAtIso }

## Notes

- Loop cadence is controlled by the interval-length node in the subflow (default 30s).
- Influx retention policy assumed autogen/infinite; adjust on DB if desired.
- When sharing this flow, replace broker/DB configs or use placeholders.

## Shelly Pro 3EM Setup

To allow historical polling via MQTT RPC (EMData.GetData / EMData.GetRecords), configure the device as follows.

Shelly UI: **Device → Settings → Network**

- MQTT **enabled**
- RPC status notifications over MQTT **on**  
- Generic status update over MQTT **on**  
- SSL connectivity **on** (optional, but recommended)  
- Select certificate (if using SSL)  
- MQTT Broker host / port  
- MQTT Client ID – must match `SHELLY_ID` ENV var  
- MQTT Broker username / password

Once configured and saved, the device will accept RPC requests over MQTT and return stored history when available.