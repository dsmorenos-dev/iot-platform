## 🔄 High-Level Flow

```mermaid
flowchart LR
    subgraph Device[ESP32 Device]
        S1[Read sensors]
        S2[Format telemetry JSON]
        S3[Publish telemetry via MQTT/HTTP]
        S4[Receive control commands]
        S5[Actuate outputs (relays, LEDs, etc.)]
    end

    subgraph Backend[Backend API (FastAPI)]
        B1[Receive telemetry<br/>(MQTT subscriber / HTTP endpoint)]
        B2[Validate & normalize data]
        B3[Store in PostgreSQL]
        B4[Expose REST & WebSocket endpoints]
        B5[Persist commands to DB]
        B6[Publish commands via MQTT]
    end

    subgraph DB[(PostgreSQL)]
        D1[(measurements)]
        D2[(devices)]
        D3[(commands)]
        D4[(users)]
    end

    subgraph Services[Background Services (Python)]
        SV1[Data processor<br/>(aggregations, thresholds)]
        SV2[Alerts service<br/>(email/notifications)]
    end

    subgraph Frontend[Frontend Dashboard (Next.js)]
        F1[User logs in<br/>(JWT auth)]
        F2[Request device list & status]
        F3[Subscribe to live data<br/>(WebSocket)]
        F4[View charts & history]
        F5[Send control command<br/>(REST call)]
    end

    %% Telemetry path
    S1 --> S2 --> S3 --> B1 --> B2 --> B3 --> D1
    B3 --- D2
    B3 --- D3
    B3 --- D4

    %% Services path
    D1 --> SV1 --> SV2

    %% Frontend data path
    F1 --> B4
    F2 --> B4
    F3 --> B4
    B4 --> D1
    B4 --> D2
    B4 --> D3
    B4 --> D4
    D1 --> F4

    %% Command path
    F5 --> B5 --> D3 --> B6 --> S4 --> S5


```markdown
## 📡 Sequence Diagram – Telemetry from Device to Dashboard

```mermaid
sequenceDiagram
    participant ESP as ESP32 Device
    participant MQTT as MQTT Broker
    participant API as Backend API (FastAPI)
    participant DB as PostgreSQL
    participant FE as Frontend (Next.js)

    Note over ESP: Device boots<br/>and connects to Wi-Fi
    ESP->>MQTT: CONNECT + AUTH
    MQTT-->>ESP: CONNACK

    loop Periodic telemetry
        ESP->>MQTT: PUBLISH telemetry<br/>topic: devices/{id}/telemetry
        MQTT->>API: Forward message<br/>(MQTT subscription)
        API->>API: Validate & parse payload
        API->>DB: INSERT INTO measurements
        DB-->>API: ACK

        par Update dashboards
            API-->>FE: WebSocket push<br/>new measurement
            FE->>FE: Update UI charts/state
        and Optional processing
            API-->>API: Emit internal event<br/>for background services
        end
    end


```markdown
## 🧭 Sequence Diagram – User Sends Command to Device

```mermaid
sequenceDiagram
    participant User as User
    participant FE as Frontend (Next.js)
    participant API as Backend API (FastAPI)
    participant DB as PostgreSQL
    participant MQTT as MQTT Broker
    participant ESP as ESP32 Device

    User->>FE: Click "Turn ON relay"
    FE->>API: POST /devices/{id}/commands<br/>{ type: "RELAY_ON" }
    API->>API: Validate auth & payload
    API->>DB: INSERT INTO commands
    DB-->>API: Command stored

    API->>MQTT: PUBLISH command<br/>topic: devices/{id}/commands
    MQTT-->>ESP: Deliver command message

    ESP->>ESP: Parse command JSON<br/>execute action (RELAY_ON)
    ESP->>MQTT: PUBLISH updated telemetry<br/>(state = "ON")
    MQTT->>API: Telemetry message
    API->>DB: INSERT new measurement/state
    API-->>FE: WebSocket event<br/>{ state: "ON" }
    FE->>User: Update UI (relay is ON)