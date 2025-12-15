# PowerEye Protocol Documentation

## Overview

The PowerEye Protocol defines the protocols that a device should implement to communicate with the PowerEye Application via MQTT pub/sub channels.

## 1. MQTT Connection Information

### 1.1 Root CA Certificate

```
rootCA: string
```

Certificate Authority (CA) root certificate to authenticate a secure SSL/TLS connection.

### 1.2 Broker Information

#### Host

```
host: string
```

The IP address or domain name of the MQTT broker.

**Example:**

- `mqtt.powereye.co`
- `192.168.1.100`

#### Port

```
port: number
```

The connection port for the MQTT broker.

**Common Ports:**

- `8883` - MQTT with SSL/TLS

### 1.3 Client Information

#### Prefix Topic

```
prefixTopic: string
```

The prefix for all MQTT topics of this device.

**Format:** `{client_id}/{host}/{device_id}`

**Meaning:**

- `client_id`: The client's ID in PowerEye
- `host`: The broker identifier the device will connect to—meaning the device will connect directly to the `host` via the broker without any other intermediaries.
- `device_id`: The device's ID

**Example:** `client123/host/PE001`

#### Client Certificate

```
cert: string
```

The client's certificate for mutual TLS authentication.

#### Private Key

```
privateKey: string
```

The private key corresponding to the client certificate.

#### Username

```
username: string
```

The MQTT client username.

#### Password

```
password: string
```

The MQTT client password.

### 1.4 MQTT Topics Structure

```
{prefixTopic}/status // Will Topic

{prefixTopic}/command/request/{requestId} // Sub: Subscribe to receive commands from the application
{prefixTopic}/command/response/{requestId} // Pub: Publish command responses to the application

{prefixTopic}/value // Pub: Publish all read data from the device in a packet to the application
{prefixTopic}/value/ack // Sub: Subscribe to receive the last data recorded on the application

{prefixTopic}/param/{paramName} // Pub: Publish data changes for a single parameter to the application

```

---

## 2. MQTT Commands (Pub/Sub channels)

This section describes the topics and message formats that the device supports.

### 2.1 Status

Purpose: Provide a compact device presence indicator using MQTT Last Will and Testament (LWT).

Topic: `{prefixTopic}/status` — device publishes and sets LWT on this topic.

Behavior:

- When connecting the device MUST set an LWT on `{prefixTopic}/status` with payload `"offline"`, `QoS=1`, and `retain=true`.
- Immediately after a successful connection the device MUST publish `"online"` to `{prefixTopic}/status` with `QoS=1` and `retain=true`.
- On graceful disconnect the device SHOULD publish `"offline"` before disconnecting. On ungraceful disconnect the broker will publish the LWT (`"offline"`).

Payload: Always a simple string: either `"online"` or `"offline"`. Devices MUST NOT publish other payload formats on this topic.

Examples:

- LWT (set on connect): topic `{prefixTopic}/status`, payload: `offline`, qos: 1, retain: true
- On connect publish: topic `{prefixTopic}/status`, payload: `online`, qos: 1, retain: true

Recommendation: Use `QoS=1` and `retain=true` so the latest device state is available to subscribers.

### 2.2 Write Request

Purpose: receive a write command from the application, execute the write operation, and respond with the result to the application.

- Request channel (subscribe): `{prefixTopic}/write/request/{requestId}`
- Response channel (publish): `{prefixTopic}/write/response/{requestId}`
- The request/response topics will use `QoS=2` and `retain=false`.

Example request data format:

```json
{
  "requestId": "string",
  "data": { "param": "Temperature", "value": 32.6 }
}
```

Example successful response format:

```json
{
  "requestId": "string",
  "data": {
    "Temperature": 32.6
  }
}
```

Example failed response format:

```json
{
  "requestId": "string",
  "error": "string"
}
```

- The `requestId` in topics, payload of request and response MUST exactly the same, enabling clients to reliably correlate responses with their requests.

### 2.3 Read Request

Purpose: receive a read command from the application, read the current value, and return it to the application.

- Request topic (subscribe): `{prefixTopic}/read/request/{requestId}`
- Response topic (publish): `{prefixTopic}/read/response/{requestId}`
- The request/response topics will use `QoS=2` and `retain=false`.

Example request format:

```json
{
  "requestId": "string",
  "data": { "params": ["Temperature", "Humidity"] }
}
```

Example response format:

```json
{
  "requestId": "string",
  "data": {
    "time": 1765177373000, // Timestamp in milliseconds - this field is optional
    "data": {
      "Temperature": 33.5,
      "Humidity": 66.9
    }
  }
}
```

Example failed response format:

```json
{
  "requestId": "string",
  "error": "string"
}
```

- The `requestId` in topics, payload of request and response MUST exactly the same, enabling clients to reliably correlate responses with their requests.

### 2.4 Data Synchronization

Purpose: synchronize data from the device to the application

#### 2.4.1 Latest value

Purpose: to know the current data stored by the application to perform the latest data synchronization.

- Latest value topic (subscribe): `{prefixTopic}/values/latest`

- The latest value topic will use `QoS=2` and `retain=true`.

- Example of received data

```json
{
  "time": 1765177373000, // Timestamp in milliseconds
  "id": 0 // Unique identifier of latest data
}
```

- When the application has no data, it will send a value of `"0"` in this command.

#### 2.4.2 Values

Purpose: synchronize data to the application.

- Data channel (publish): `{prefixTopic}/values`

- The data topic will use QoS 2 and retain=true.

Example of data sent:

```json
[
  {
    "id": 0, // Unique identifier of the record created by the device for synchronization — may be a string or a number
    "time": 1765177373000, // Timestamp in milliseconds - this field is required
    "data": {
      "Temperature": 33.5,
      "Humidity": 66.9
    }
  },
  {
    "id": 1,
    "time": 1765177375000,
    "data": {
      "Temperature": 33.5,
      "Humidity": 66.9
    }
  }
]
```

- The value in the payload is an array of reading values sorted in ascending order by time; **the last value** will be used as a reference to be sent back in value/ack.

#### 2.5 Value update

Purpose: store current value to application

- Data channel (publish): `{prefixTopic}/value`

- The data topic will use QoS 2 and retain=true.

Example of data sent:

```json
{
  "time": 1765177373000, // Timestamp in milliseconds - this field is optional
  "data": {
    "Temperature": 33.5,
    "Humidity": 66.9
  }
}
```

### 2.5 Parameter update

Purpose: update the latest value from the device to the application.

- Parameter data topic (publish): `{prefixTopic}/param/{param}`

- `{param}` is the name of the read parameter (e.g., `Temperature`, `Humidity`...). These parameters are defined by the device and declared in the application.

- The parameter data topic uses `QoS=0` or `Qos=1` and `retain=true`.

Example of updating temperature at a specific time:

`{prefixTopic}/param/Temperature`

```json
{
  "time": 1765177373000, // Timestamp in milliseconds - this field is optional
  "value": 26.9
}
```

---

## 3. Implementation Guide

### Support control device

- MUST implement the `Write Request` (see Section 2.1).

### Simple active-reading device (event-driven telemetry)

- Device can choose to implement option 1 or 2 or both:
  1. Implement parameter update using `{prefixTopic}/param/{param}` to publish single-parameter changes as they occur. The application will cache the value and store it periodically as defined in the app configuration.
  2. Implement value update using `{prefixTopic}/value` to store value in application
- Other APIs are optional and may be ignored.

### Simple passive-reading device (on-demand)

- MUST implement `Read Request` (see Section 2.1).
- Other APIs are optional and may be ignored.

### Historic-data device (historical data synchronization)

- Implement `Values` and `Latest value` (`{prefixTopic}/values` and `{prefixTopic}/values/latest`).
- Device SHOULD subscribe to `{prefixTopic}/values/latest` and reconcile the last acknowledged `id`/`time` before publishing historical batches to `{prefixTopic}/values`.
- When publishing historical data, send an ordered array of records (ascending by `time`) so the application can process them sequentially.

---

---

**Version:** 0.0.3  
**Last Updated:** December 15, 2025
