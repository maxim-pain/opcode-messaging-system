# OPCODE Messaging System - Advanced Visualization and Integration

*Date: 2025-08-12*

## 1. Comprehensive End-to-End Configuration Mapping

This detailed visualization shows the complete journey from OPCODE definition to final message delivery, with all configuration components mapped to their roles in the process.

```mermaid
flowchart TB
    %% Configuration Elements
    subgraph ConfigElements["Configuration Elements"]
        direction TB
        CONFIG_FILE["JSON Configuration Files"]
        OPCODE_DEF["OPCODE Definitions"]
        DRIVER_DEF["Driver Definitions"]
        PROTOCOL_DEF["Protocol Definitions"]
        ROUTE_DEF["Route Definitions"]
        QOS_DEF["QoS Profiles"]

        CONFIG_FILE --> OPCODE_DEF
        CONFIG_FILE --> DRIVER_DEF
        CONFIG_FILE --> PROTOCOL_DEF
        CONFIG_FILE --> ROUTE_DEF
        CONFIG_FILE --> QOS_DEF
    end

    %% Runtime Components
    subgraph RuntimeComponents["Runtime Components"]
        direction TB
        CM["CommManager"]
        OPREG["OpcodeRegistry"]
        ROUTE_MGR["RouteManager"]
        PROTO_FACT["ProtocolFactory"]
        DRV_MGR["DriverManager"]
        QOS_MGR["QoSManager"]

        CM --> OPREG
        CM --> ROUTE_MGR
        CM --> PROTO_FACT
        CM --> DRV_MGR
        CM --> QOS_MGR
    end

    %% Configuration to Runtime Mapping
    OPCODE_DEF --> OPREG
    DRIVER_DEF --> DRV_MGR
    PROTOCOL_DEF --> PROTO_FACT
    ROUTE_DEF --> ROUTE_MGR
    QOS_DEF --> QOS_MGR

    %% Message Flow
    APP_PUB["Application: publish()"]
    APP_SUB["Application: subscribe()"]

    APP_PUB --> CM
    CM --> APP_SUB

    %% Extended Flow with Message Operations
    subgraph MessageOps["Message Operations"]
        direction TB
        MSG_CREATE["1. Create Message<br/>with OPCODE"]
        OPCODE_VAL["2. Validate OPCODE<br/>Check Max Size"]
        ROUTE_RES["3. Resolve Route<br/>Find Subscribers"]
        QOS_APPLY["4. Apply QoS<br/>Prioritize Message"]
        PROTO_ENC["5. Protocol Encoding<br/>Add Headers, Fragment"]
        DRIVER_TX["6. Driver Transmission<br/>Physical Layer Sending"]

        MSG_CREATE --> OPCODE_VAL
        OPCODE_VAL --> ROUTE_RES
        ROUTE_RES --> QOS_APPLY
        QOS_APPLY --> PROTO_ENC
        PROTO_ENC --> DRIVER_TX
    end

    APP_PUB --> MSG_CREATE

    %% Component to Operation Mapping
    OPREG -.-> OPCODE_VAL
    ROUTE_MGR -.-> ROUTE_RES
    QOS_MGR -.-> QOS_APPLY
    PROTO_FACT -.-> PROTO_ENC
    DRV_MGR -.-> DRIVER_TX

    %% Message Reception Flow
    subgraph MessageRec["Message Reception"]
        direction TB
        DRV_RX["1. Driver Reception<br/>Physical Layer Receiving"]
        PROTO_DEC["2. Protocol Decoding<br/>Parse Headers, Reassemble"]
        ROUTE_FWD["3. Routing Decision<br/>Forward or Deliver"]
        MSG_DLVR["4. Message Delivery<br/>To Subscribers"]
        ACK_GEN["5. ACK Generation<br/>If Required"]

        DRV_RX --> PROTO_DEC
        PROTO_DEC --> ROUTE_FWD
        ROUTE_FWD -->|"Deliver"| MSG_DLVR
        ROUTE_FWD -->|"Forward"| PROTO_ENC
        MSG_DLVR --> ACK_GEN
    end

    DRIVER_TX --> DRV_RX
    MSG_DLVR --> APP_SUB

    %% Style nodes
    classDef config fill:#f9f,stroke:#333,stroke-width:1px
    classDef runtime fill:#ff9,stroke:#333,stroke-width:2px
    classDef app fill:#9cf,stroke:#333,stroke-width:1px
    classDef operation fill:#cfc,stroke:#333,stroke-width:1px

    class CONFIG_FILE,OPCODE_DEF,DRIVER_DEF,PROTOCOL_DEF,ROUTE_DEF,QOS_DEF config
    class CM,OPREG,ROUTE_MGR,PROTO_FACT,DRV_MGR,QOS_MGR runtime
    class APP_PUB,APP_SUB app
    class MSG_CREATE,OPCODE_VAL,ROUTE_RES,QOS_APPLY,PROTO_ENC,DRIVER_TX,DRV_RX,PROTO_DEC,ROUTE_FWD,MSG_DLVR,ACK_GEN operation
```

## 2. Advanced Visualization: Multi-SoC OPCODE Routing

This visualization demonstrates how OPCODEs are routed across a complex multi-SoC system with different transport layers.

```mermaid
graph TD
    %% SoCs and Cores
    subgraph SoC1["SoC1 - Application Processor"]
        SoC1_ARM["ARM Core (Application)"]
        SoC1_DSP["DSP Core (Audio Processing)"]

        SoC1_ARM ---|"OpenAMP IPC"| SoC1_DSP
    end

    subgraph SoC2["SoC2 - Connectivity"]
        SoC2_BLE["BLE Core"]
    end

    subgraph SoC3["SoC3 - Sensor Hub"]
        SoC3_MCU["MCU Core (Sensor Processing)"]
    end

    %% Inter-SoC Connections
    SoC1_ARM ---|"UART"| SoC2_BLE
    SoC1_ARM ---|"I2C"| SoC3_MCU

    %% OPCODE Routing Visualization
    SYS_OPCODES["System OPCODEs<br/>(0x0001-0x00FF)"]
    AUDIO_OPCODES["Audio OPCODEs<br/>(0x0100-0x01FF)"]
    BLE_OPCODES["BLE OPCODEs<br/>(0x0200-0x02FF)"]
    SENSOR_OPCODES["Sensor OPCODEs<br/>(0x0300-0x03FF)"]

    %% OPCODE to Core Subscriptions
    SYS_OPCODES -->|"Subscribe"| SoC1_ARM
    SYS_OPCODES -->|"Subscribe"| SoC2_BLE
    SYS_OPCODES -->|"Subscribe"| SoC3_MCU

    AUDIO_OPCODES -->|"Subscribe"| SoC1_DSP
    AUDIO_OPCODES -->|"Limited Subscribe"| SoC1_ARM

    BLE_OPCODES -->|"Primary Subscribe"| SoC2_BLE
    BLE_OPCODES -->|"Status Subscribe"| SoC1_ARM

    SENSOR_OPCODES -->|"Primary Subscribe"| SoC3_MCU
    SENSOR_OPCODES -->|"Processed Data Subscribe"| SoC1_ARM
    SENSOR_OPCODES -->|"Motion Events Subscribe"| SoC1_DSP

    %% Example Message Flows
    MSG_SYS["SYS_POWER_STATE<br/>OPCODE 0x0002"]
    MSG_SYS -->|"Publish"| SoC1_ARM

    MSG_AUDIO["AUDIO_EQ_SETTINGS<br/>OPCODE 0x0101"]
    MSG_AUDIO -->|"Publish"| SoC1_ARM

    MSG_BLE["BLE_CONNECTION_STATE<br/>OPCODE 0x0200"]
    MSG_BLE -->|"Publish"| SoC2_BLE

    MSG_SENSOR["SENSOR_ACCEL_DATA<br/>OPCODE 0x0300"]
    MSG_SENSOR -->|"Publish"| SoC3_MCU

    %% Routing Paths with Protocols
    SoC1_ARM -->|"custom_ack, HIGH"| SoC1_DSP
    SoC1_ARM -->|"secure_protocol, HIGH"| SoC2_BLE
    SoC1_ARM -->|"custom_ack, NORMAL"| SoC3_MCU

    SoC2_BLE -->|"secure_protocol, HIGH"| SoC1_ARM

    SoC3_MCU -->|"custom_ack, LOW"| SoC1_ARM
    SoC1_ARM -->|"custom_lite, NORMAL"| SoC1_DSP

    %% Style nodes
    classDef socNode fill:#ffcccc,stroke:#333,stroke-width:2px
    classDef coreNode fill:#ccffcc,stroke:#333,stroke-width:1px
    classDef opcodeNode fill:#ccccff,stroke:#333,stroke-width:1px
    classDef msgNode fill:#ffffcc,stroke:#333,stroke-width:1px

    class SoC1_ARM,SoC1_DSP,SoC2_BLE,SoC3_MCU coreNode
    class SYS_OPCODES,AUDIO_OPCODES,BLE_OPCODES,SENSOR_OPCODES opcodeNode
    class MSG_SYS,MSG_AUDIO,MSG_BLE,MSG_SENSOR msgNode
```

## 3. Protocol Stack Visualization

This diagram illustrates the layered protocol architecture and how different protocol variants handle message encapsulation.

```mermaid
graph TD
    subgraph AppLayer["Application Layer"]
        APP_MSG["Application Message<br/>{OPCODE, payload}"]
    end

    subgraph ProtoSelection["Protocol Selection"]
        PROTO_SELECTOR["Protocol Selector<br/>Based on OPCODE, Route, QoS"]

        CUSTOM_ACK["CUSTOM_ACK Protocol<br/>Reliability-focused"]
        CUSTOM_LITE["CUSTOM_LITE Protocol<br/>Efficiency-focused"]
        SECURE_AES["SECURE_AES Protocol<br/>Security-focused"]

        PROTO_SELECTOR --> CUSTOM_ACK
        PROTO_SELECTOR --> CUSTOM_LITE
        PROTO_SELECTOR --> SECURE_AES
    end

    subgraph ProtoEncoding["Protocol Encoding Layers"]
        subgraph CustomAckLayers["CUSTOM_ACK Layers"]
            ACK_BASE["Base Layer<br/>opcode, length, seqNum"]
            ACK_RELIABILITY["Reliability Layer<br/>token, retryCount, flags"]
            ACK_INTEGRITY["Integrity Layer<br/>CRC-16"]

            ACK_BASE --> ACK_RELIABILITY
            ACK_RELIABILITY --> ACK_INTEGRITY
        end

        subgraph CustomLiteLayers["CUSTOM_LITE Layers"]
            LITE_BASE["Base Layer<br/>compressed header"]
            LITE_INTEGRITY["Optional Integrity<br/>CRC-8"]

            LITE_BASE --> LITE_INTEGRITY
        end

        subgraph SecureAesLayers["SECURE_AES Layers"]
            SEC_BASE["Base Layer<br/>opcode, length, seqNum"]
            SEC_AUTH["Authentication<br/>HMAC"]
            SEC_ENCRYPT["Encryption Layer<br/>AES-256"]
            SEC_INTEGRITY["Integrity Layer<br/>CRC-16"]

            SEC_BASE --> SEC_AUTH
            SEC_AUTH --> SEC_ENCRYPT
            SEC_ENCRYPT --> SEC_INTEGRITY
        end
    end

    subgraph FrameFormats["Frame Formats"]
        ACK_FRAME["CUSTOM_ACK Frame<br/>|header|token|payload|CRC|"]
        LITE_FRAME["CUSTOM_LITE Frame<br/>|comp. header|payload|CRC|"]
        SEC_FRAME["SECURE_AES Frame<br/>|header|IV|enc. payload|HMAC|CRC|"]

        ACK_INTEGRITY --> ACK_FRAME
        LITE_INTEGRITY --> LITE_FRAME
        SEC_INTEGRITY --> SEC_FRAME
    end

    subgraph TransportLayer["Transport Layer"]
        DRIVER_TX["Driver Transmission<br/>Physical Layer Constraints"]

        ACK_FRAME --> DRIVER_TX
        LITE_FRAME --> DRIVER_TX
        SEC_FRAME --> DRIVER_TX
    end

    %% Message flow
    APP_MSG --> PROTO_SELECTOR

    %% Style nodes
    classDef appLayer fill:#ffcccc,stroke:#333,stroke-width:1px
    classDef protoSelect fill:#ccffcc,stroke:#333,stroke-width:1px
    classDef protoLayer fill:#ccccff,stroke:#333,stroke-width:1px
    classDef frameFormat fill:#ffffcc,stroke:#333,stroke-width:1px
    classDef transport fill:#ffccff,stroke:#333,stroke-width:1px

    class APP_MSG appLayer
    class PROTO_SELECTOR,CUSTOM_ACK,CUSTOM_LITE,SECURE_AES protoSelect
    class ACK_BASE,ACK_RELIABILITY,ACK_INTEGRITY,LITE_BASE,LITE_INTEGRITY,SEC_BASE,SEC_AUTH,SEC_ENCRYPT,SEC_INTEGRITY protoLayer
    class ACK_FRAME,LITE_FRAME,SEC_FRAME frameFormat
    class DRIVER_TX transport
```

## 4. QoS Management Visualization

This diagram shows how QoS policies are applied to messages based on priority and route characteristics.

```mermaid
graph TD
    %% Message Creation with QoS
    MSG_CREATE["New Message<br/>{OPCODE, payload, priority}"]

    subgraph QoSClassification["QoS Classification"]
        QOS_CLASSIFY["QoS Classification<br/>Based on OPCODE, Priority"]

        QOS_URGENT["URGENT QoS<br/>Realtime, Low Latency"]
        QOS_HIGH["HIGH QoS<br/>Guaranteed Delivery"]
        QOS_NORMAL["NORMAL QoS<br/>Best Effort"]
        QOS_LOW["LOW QoS<br/>Background"]

        QOS_CLASSIFY --> QOS_URGENT
        QOS_CLASSIFY --> QOS_HIGH
        QOS_CLASSIFY --> QOS_NORMAL
        QOS_CLASSIFY --> QOS_LOW
    end

    subgraph ResourceManagement["Resource Management"]
        RES_ALLOC["Resource Allocation<br/>Based on QoS Profile"]

        CPU_ALLOC["CPU Resources<br/>Thread Priority"]
        MEM_ALLOC["Memory Allocation<br/>Dedicated Pools"]
        BW_ALLOC["Bandwidth Allocation<br/>Transmission Slots"]

        RES_ALLOC --> CPU_ALLOC
        RES_ALLOC --> MEM_ALLOC
        RES_ALLOC --> BW_ALLOC
    end

    subgraph QueueManagement["Queue Management"]
        QUEUE_MGR["Priority Queue Manager"]

        QUEUE_U["URGENT Queue<br/>Preemptive"]
        QUEUE_H["HIGH Queue"]
        QUEUE_N["NORMAL Queue"]
        QUEUE_L["LOW Queue"]

        QUEUE_MGR --> QUEUE_U
        QUEUE_MGR --> QUEUE_H
        QUEUE_MGR --> QUEUE_N
        QUEUE_MGR --> QUEUE_L
    end

    subgraph Scheduler["Scheduler"]
        SCHED["Queue Scheduler<br/>Deficit Round-Robin"]

        SCHED_U["URGENT Scheduler<br/>Weight: 16"]
        SCHED_H["HIGH Scheduler<br/>Weight: 8"]
        SCHED_N["NORMAL Scheduler<br/>Weight: 4"]
        SCHED_L["LOW Scheduler<br/>Weight: 1"]

        SCHED --> SCHED_U
        SCHED --> SCHED_H
        SCHED --> SCHED_N
        SCHED --> SCHED_L
    end

    subgraph Transmission["Transmission"]
        TX_MGR["Transmission Manager"]

        TX_IMMEDIATE["Immediate Transmission<br/>URGENT Messages"]
        TX_RELIABLE["Reliable Transmission<br/>HIGH/NORMAL Messages"]
        TX_BACKGROUND["Background Transmission<br/>LOW Messages"]

        TX_MGR --> TX_IMMEDIATE
        TX_MGR --> TX_RELIABLE
        TX_MGR --> TX_BACKGROUND
    end

    %% Message flow
    MSG_CREATE --> QOS_CLASSIFY

    QOS_URGENT --> RES_ALLOC
    QOS_HIGH --> RES_ALLOC
    QOS_NORMAL --> RES_ALLOC
    QOS_LOW --> RES_ALLOC

    QOS_URGENT --> QUEUE_U
    QOS_HIGH --> QUEUE_H
    QOS_NORMAL --> QUEUE_N
    QOS_LOW --> QUEUE_L

    QUEUE_U --> SCHED_U
    QUEUE_H --> SCHED_H
    QUEUE_N --> SCHED_N
    QUEUE_L --> SCHED_L

    SCHED_U --> TX_IMMEDIATE
    SCHED_H --> TX_RELIABLE
    SCHED_N --> TX_RELIABLE
    SCHED_L --> TX_BACKGROUND

    TX_IMMEDIATE --> MSG_SEND["Message Transmission"]
    TX_RELIABLE --> MSG_SEND
    TX_BACKGROUND --> MSG_SEND

    %% Style nodes
    classDef msgCreate fill:#ffcccc,stroke:#333,stroke-width:1px
    classDef qosClass fill:#ccffcc,stroke:#333,stroke-width:1px
    classDef resAlloc fill:#ccccff,stroke:#333,stroke-width:1px
    classDef queueMgr fill:#ffffcc,stroke:#333,stroke-width:1px
    classDef scheduler fill:#ffccff,stroke:#333,stroke-width:1px
    classDef transmission fill:#ccffff,stroke:#333,stroke-width:1px

    class MSG_CREATE,MSG_SEND msgCreate
    class QOS_CLASSIFY,QOS_URGENT,QOS_HIGH,QOS_NORMAL,QOS_LOW qosClass
    class RES_ALLOC,CPU_ALLOC,MEM_ALLOC,BW_ALLOC resAlloc
    class QUEUE_MGR,QUEUE_U,QUEUE_H,QUEUE_N,QUEUE_L queueMgr
    class SCHED,SCHED_U,SCHED_H,SCHED_N,SCHED_L scheduler
    class TX_MGR,TX_IMMEDIATE,TX_RELIABLE,TX_BACKGROUND transmission
```

## 5. End-to-End System Integration View

This comprehensive visualization shows the complete system with all components and their interactions, from configuration through runtime execution.

```mermaid
graph TB
    %% Top-level system overview
    subgraph SystemConfigurationLayer["System Configuration Layer"]
        CONFIG_FILES["Configuration Files<br/>JSON/YAML"]
        CONFIG_MGR["Configuration Manager<br/>Validation, Hot-swap"]

        CONFIG_FILES --> CONFIG_MGR
    end

    subgraph RuntimeSystem["Runtime System"]
        COMM_MGR["Communication Manager<br/>Central Hub"]

        subgraph ComponentManagers["Component Managers"]
            OP_REG["OPCODE Registry<br/>Metadata, Validation"]
            ROUTE_MGR["Route Manager<br/>Path Resolution"]
            PROTO_MGR["Protocol Manager<br/>Encoding/Decoding"]
            DRIVER_MGR["Driver Manager<br/>Transport Handling"]
            QOS_MGR["QoS Manager<br/>Prioritization"]
        end

        COMM_MGR <--> OP_REG
        COMM_MGR <--> ROUTE_MGR
        COMM_MGR <--> PROTO_MGR
        COMM_MGR <--> DRIVER_MGR
        COMM_MGR <--> QOS_MGR
    end

    CONFIG_MGR -->|Configure| COMM_MGR
    CONFIG_MGR -->|Configure| OP_REG
    CONFIG_MGR -->|Configure| ROUTE_MGR
    CONFIG_MGR -->|Configure| PROTO_MGR
    CONFIG_MGR -->|Configure| DRIVER_MGR
    CONFIG_MGR -->|Configure| QOS_MGR

    subgraph ApplicationInterface["Application Interface"]
        APP_API["Application API<br/>publish(), subscribe()"]
        APP_API <-->|Use| COMM_MGR
    end

    subgraph HardwareIntegrationLayer["Hardware Integration Layer"]
        IPC_HW["IPC Hardware<br/>Shared Memory, Mailboxes"]
        I2C_HW["I2C Hardware<br/>Controllers, Buses"]
        SPI_HW["SPI Hardware<br/>Controllers, Buses"]
        UART_HW["UART Hardware<br/>Serial Ports"]

        DRIVER_MGR -->|Access| IPC_HW
        DRIVER_MGR -->|Access| I2C_HW
        DRIVER_MGR -->|Access| SPI_HW
        DRIVER_MGR -->|Access| UART_HW
    end

    subgraph PhysicalTopology["Physical Topology"]
        SOC1["SoC 1<br/>Application Processor"]
        SOC2["SoC 2<br/>Connectivity"]
        SOC3["SoC 3<br/>Sensor Hub"]

        SOC1 <-->|IPC| SOC1
        SOC1 <-->|UART| SOC2
        SOC1 <-->|I2C| SOC3
    end

    IPC_HW -->|Connect| SOC1
    UART_HW -->|Connect| SOC2
    I2C_HW -->|Connect| SOC3

    %% Message Flow Overlay
    MSG_FLOW_1["1. App Creates Message"]
    MSG_FLOW_2["2. Validate OPCODE"]
    MSG_FLOW_3["3. Apply QoS"]
    MSG_FLOW_4["4. Resolve Routes"]
    MSG_FLOW_5["5. Encode Protocol"]
    MSG_FLOW_6["6. Transmit via Driver"]
    MSG_FLOW_7["7. Hardware Transmission"]

    MSG_FLOW_1 -->|Flow| APP_API
    APP_API -->|Flow| MSG_FLOW_2
    MSG_FLOW_2 -->|Flow| OP_REG
    OP_REG -->|Flow| MSG_FLOW_3
    MSG_FLOW_3 -->|Flow| QOS_MGR
    QOS_MGR -->|Flow| MSG_FLOW_4
    MSG_FLOW_4 -->|Flow| ROUTE_MGR
    ROUTE_MGR -->|Flow| MSG_FLOW_5
    MSG_FLOW_5 -->|Flow| PROTO_MGR
    PROTO_MGR -->|Flow| MSG_FLOW_6
    MSG_FLOW_6 -->|Flow| DRIVER_MGR
    DRIVER_MGR -->|Flow| MSG_FLOW_7
    MSG_FLOW_7 -->|Flow| IPC_HW

    %% Style nodes
    classDef configLayer fill:#ffdddd,stroke:#333,stroke-width:2px
    classDef runtimeSys fill:#ddffdd,stroke:#333,stroke-width:2px
    classDef compMgr fill:#ddddff,stroke:#333,stroke-width:1px
    classDef appIface fill:#ffffdd,stroke:#333,stroke-width:2px
    classDef hwLayer fill:#ddffff,stroke:#333,stroke-width:2px
    classDef topology fill:#ffddff,stroke:#333,stroke-width:2px
    classDef msgFlow fill:#ffffff,stroke:#ff0000,stroke-width:1px,stroke-dasharray: 5 5

    class CONFIG_FILES,CONFIG_MGR configLayer
    class COMM_MGR runtimeSys
    class OP_REG,ROUTE_MGR,PROTO_MGR,DRIVER_MGR,QOS_MGR compMgr
    class APP_API appIface
    class IPC_HW,I2C_HW,SPI_HW,UART_HW hwLayer
    class SOC1,SOC2,SOC3 topology
    class MSG_FLOW_1,MSG_FLOW_2,MSG_FLOW_3,MSG_FLOW_4,MSG_FLOW_5,MSG_FLOW_6,MSG_FLOW_7 msgFlow
```

This comprehensive set of visualizations provides a clear understanding of how the entire OPCODE messaging system works together, from configuration through runtime message processing, across multiple SoCs and transport layers.
