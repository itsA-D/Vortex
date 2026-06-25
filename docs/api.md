# 🔌 API Specification & Protocol Guide

This document defines the web interface APIs, the WebSocket sub-protocol payloads, and the TCP socket command interface used by the **NExSyS Simulation Monitor Dashboard**.

---

## 1. HTTP Endpoints

The Go backend serves a dual purpose as an HTTP asset host and a WebSocket bridge.

### 1.1 Web Asset Host
* **Endpoint:** `GET /`
* **Response:** Returns the optimized React build bundle index (`index.html`) and associated static JS/CSS assets from the local `./build/` directory.

### 1.2 WebSocket Upgrade
* **Endpoint:** `GET /ws`
* **Protocol:** Upgrades the connection from HTTP/1.1 to RFC 6455 WebSockets.
* **Headers Required:**
  ```http
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Key: <Client-Generated-Key>
  Sec-WebSocket-Version: 13
  ```

---

## 2. WebSocket Protocol Schema

All WebSocket communication uses UTF-8 JSON text payloads. Telemetry updates and control commands flow bidirectionally.

### 2.1 Client-to-Server Commands (Browser → Go Broker)

#### A. Active Panel Change Command
Sent by the React client when a user navigates to a new dashboard subsystem page. This tells the backend to retrieve and start transmitting only the variables mapped to that specific subsystem.

- **Payload Structure:**
  ```json
  {
    "variable": "activePanel",
    "value": "PANEL_IDENTIFIER"
  }
  ```
- **Supported Panel Identifiers:**
  - `"dashboard"` (Default overview panel)
  - `"mpcv"` (Guidance, Navigation, and Control)
  - `"pm"` (Power Management systems)
  - `"robo"` (Robotics arm interfaces)
  - `"subsys"` (Cabin environmental status)
  - `"cams"` (Telemetry telescopes and cameras)
  - `"rover_llt"` (Rover status)

#### B. Trick Server Reconnect Command
Sent by the client when configuring the dashboard to stream telemetry from a different Trick simulation server.

- **Payload Structure:**
  ```json
  {
    "variable": "trickServer",
    "value": "IP_OR_HOSTNAME:PORT"
  }
  ```
- **Example:**
  ```json
  {
    "variable": "trickServer",
    "value": "139.169.203.141:17000"
  }
  ```

---

### 2.2 Server-to-Client Payloads (Go Broker → Browser)

#### A. Panel Initialization payload (Map Struct)
Transmitted immediately after a successful "Active Panel Change" command. Returns a map of all variable IDs and values assigned to that specific panel. The final index always contains the confirmational panel tag.

- **Payload Structure:**
  ```json
  {
    "1": { "variable": "var_name_1", "value": "current_value_1" },
    "2": { "variable": "var_name_2", "value": "current_value_2" },
    "N": { "variable": "panel", "value": "PANEL_IDENTIFIER" }
  }
  ```
- **Example (MPCV Switch Response):**
  ```json
  {
    "1": { "variable": "dyn.inertial.position[0]", "value": "-1405.34" },
    "2": { "variable": "dyn.inertial.position[1]", "value": "4502.81" },
    "3": { "variable": "panel", "value": "mpcv" }
  }
  ```

#### B. Real-time Telemetry State Updates
Sent when variables undergo changes in the active Trick simulation run. The server sends either a single update packet or a map slice.

- **Payload Structure (Single telemetry event):**
  ```json
  {
    "variable": "dyn.inertial.position[0]",
    "value": "-1405.39"
  }
  ```

#### C. Operational Errors
Emitted by the server to inform the client of simulation connection failures.

- **Payload Structure (Invalid Trick Server Address):**
  ```json
  {
    "0": { "variable": "error", "value": "invalid_trick_server" },
    "1": { "variable": "panel", "value": "dashboard" }
  }
  ```

---

## 3. Go-to-Trick TCP Socket Interface

The communication between the Go backend and the Trick Simulation Server occurs over a raw TCP socket (default port `17000`).

### 3.1 Commands Sent by Go Backend (ASCII Statements)
The Go backend writes raw ASCII Python commands terminated with a newline (`\n`) to configure the Trick server telemetry engine:

| Command | Action | Description |
|---------|--------|-------------|
| `trick.var_pause()\n` | Pause Telemetry | Halts data streaming while configuring telemetry lists. |
| `trick.var_ascii()\n` | Set Output Encoding | Instructs Trick to output values as space-delimited text characters instead of binary structures. |
| `trick.var_add("<var_name>")\n` | Add Variable | Adds the named simulation variable to the streaming list. |
| `trick.var_unpause()\n` | Resume Telemetry | Resumes real-time streaming over the socket. |

- **Example Registration Stream:**
  ```python
  trick.var_pause()
  trick.var_ascii()
  trick.var_add("dyn.inertial.position[0]")
  trick.var_add("dyn.inertial.position[1]")
  trick.var_unpause()
  ```

### 3.2 Trick Streaming Telemetry Format
Once running, the Trick Server streams space-delimited ASCII values terminated by a newline (`\n`) on every frame execution:

- **Format:**
  ```
  <timestamp> <val_1> <val_2> <val_3> ... <val_N>\n
  ```
- **Example Data Packets received by Go:**
  ```
  10.0230000 -1405.34 4502.81 24.5 0 1.002
  10.0430000 -1405.39 4502.88 24.5 0 1.002
  ```

The Go backend utilizes `bufio.NewReader` to split packets by `\n` characters and tokenizes values using `strings.Fields(string(reply))`. The parsed index matches the sequence of variable registrations sent during startup.
