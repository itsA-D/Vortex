# ⚙️ Installation & Configuration Guide

This guide provides instructions for installing, configuring, running, and deploying the **NExSyS Simulation Monitor Dashboard**.

---

## 1. Environment Prerequisites

Before beginning, ensure that your system meets these dependency versions:

- **Go Compiler:** v1.26 or higher (configured in your `$GOPATH` or using modern Go module environments).
- **Node.js:** v14.x through v18.x recommended (due to React 16.2.0 compatibility).
- **npm:** v6.x or higher.
- **Trick Simulation System:** Installed and active on a reachable network server.

---

## 2. Step-by-Step Installation

### Step 2.1: Clone the Repository
For historic Go workspace structures, clone this repository directly into your Go workspace source folder:
```bash
cd ~/{Your_Go_Workspace}/src/
git clone https://github.com/NASA/React-GoLang-Real-time-Dashboard.git nexsysSimMonitor
cd nexsysSimMonitor
```

### Step 2.2: Install Node Dependencies
Use `npm` to download frontend packages, styling engines, and UI utilities:
```bash
npm install
```

> [!TIP]
> **Troubleshooting Sass Module Errors:**
> If you encounter compilation failures related to `node-sass` or old CSS compilers:
> 1. We have updated the styles to compile via modern Dart Sass (`sass` v1.77.1).
> 2. Ensure your build scripts in `package.json` use the pre-configured sass compiler runner:
>    ```bash
>    npm run build-css
>    ```
> 3. If dependencies clash, resolve them by cleaning node modules and forcing modern packages:
>    ```bash
>    rm -rf node_modules package-lock.json && npm install
>    ```

---

## 3. Telemetry Configuration (`vars_displayed.xml`)

Variables monitored by the dashboard are managed through a centralized XML configuration file located at [src/variables/trick/vars_displayed.xml](file:///a:/shinobi%20no%20shuriken/github%20repo/React-GoLang-Real-time-Dashboard-master/React-GoLang-Real-time-Dashboard-master/src/variables/trick/vars_displayed.xml).

### 3.1 Understanding the XML Structure
The XML schema matches the custom Go structures defined in `TrickVarParse.go`. Subsystems are split into page sections containing nested variable objects:

```xml
<FromTrick>
    <!-- MPCV Panel Section -->
    <MpcvGncPageActive>
        <MpcvGNC>
            <TrickVariable Name="dyn.inertial.position[0]" />
            <TrickVariable Name="dyn.inertial.position[1]" />
        </MpcvGNC>
        <MpcvHware></MpcvHware>
        <MpcvStates></MpcvStates>
        <MpcvGNC_simct_interface></MpcvGNC_simct_interface>
        <MpcvHC></MpcvHC>
        <TrickVariable Name="dyn.control.auto_mode" />
    </MpcvGncPageActive>

    <!-- PM Panel Section -->
    <PmGncPageActive>
        <PmGNC>
            <TrickVariable Name="pm.pdu.voltage[0]" />
        </PmGNC>
    </PmGncPageActive>
</FromTrick>
```

### 3.2 Adding a New Simulation Variable to Monitor
1. Open the [vars_displayed.xml](file:///a:/shinobi%20no%20shuriken/github%20repo/React-GoLang-Real-time-Dashboard-master/React-GoLang-Real-time-Dashboard-master/src/variables/trick/vars_displayed.xml) file.
2. Locate the subsystem XML tag corresponding to the panel you want to update (e.g., `<MpcvGncPageActive>` or `<SubsystemPageActive>`).
3. Add a new `<TrickVariable Name="your.variable.name.here" />` node.
4. **Important Deployment Step:** Rebuild the frontend UI components and restart the Go backend server to parse and stream the newly registered variables:
   ```bash
   npm run build
   # Restart your Go backend
   ```

---

## 4. Setting Up Network Addresses

### 4.1 Configuring the WebSocket Address in React
To enable the React frontend to communicate with the Go backend, you must configure the target address:
1. Open [src/variables/Variables.jsx](file:///a:/shinobi%20no%20shuriken/github%20repo/React-GoLang-Real-time-Dashboard-master/React-GoLang-Real-time-Dashboard-master/src/variables/Variables.jsx).
2. Go to line 585:
   ```javascript
   var socket = new WebSocket("ws://" + "139.169.203.141:3000" + "/ws");
   ```
3. Update `"139.169.203.141:3000"` to reflect the address and port of your active Go backend.

> [!NOTE]
> **Dynamic Host Resolution:**
> To avoid manual code updates during different environment deployments, you can uncomment line 582 to resolve the WebSocket endpoint dynamically based on the current browser host:
> ```javascript
> var socket = new WebSocket("ws://" + window.location.host + "/ws");
> ```

---

## 5. Execution Modes

### 5.1 Development Mode (Hot-Reloading UI)
Ideal for updating layouts, changing themes, and debugging SASS styles:

1. **Start the Frontend CSS Watcher & Dev Server:**
   ```bash
   npm start
   ```
   *This starts the SASS watcher and boots the React dev server at `http://localhost:3000`.*

2. **Start the Go Backend Server (Separate Terminal):**
   ```bash
   # Usage: go run server.go <host>:<port>
   go run server.go localhost:3000
   ```

### 5.2 Production Mode (Optimized & Pre-Compiled)
Ideal for standard, low-overhead deployments where the Go server hosts the entire visual stack:

1. **Compile React Production Bundle:**
   ```bash
   npm run build
   ```
   *This generates highly optimized static assets inside the `/build/` folder.*

2. **Run Server serving built files:**
   ```bash
   go run server.go localhost:3000
   ```
   *The Go backend will automatically serve static files from `/build` on HTTP requests and manage WebSocket telemetry under `/ws`.*

---

## 6. Docker Containerization (Optional Setup)

For modular containerized deployments, you can run the dashboard in an isolated environment.

Create a `Dockerfile` at the workspace root:

```dockerfile
# Step 1: Build the React static files
FROM node:16-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Step 2: Build the Go application
FROM golang:1.26-alpine
WORKDIR /app
COPY --from=builder /app/build ./build
COPY --from=builder /app/Parsers ./Parsers
COPY --from=builder /app/server.go ./
COPY --from=builder /app/go.mod ./
COPY --from=builder /app/go.sum ./
RUN go build -o main server.go

EXPOSE 3000
ENTRYPOINT ["./main", "0.0.0.0:3000"]
```

Build and run using Docker commands:
```bash
docker build -t nexsys-dashboard .
docker run -p 3000:3000 nexsys-dashboard
```
