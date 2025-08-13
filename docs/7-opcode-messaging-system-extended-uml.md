create .md file from this:

------------------------------------------------------


# OPCODE Messaging System - Extended UML Design

## Class Diagram Overview

Here's a comprehensive UML design for the OPCODE messaging system, following C++/C best practices and focusing on a clear top-down architecture:

```mermaid
classDiagram
    direction TB

    %% Core interfaces
    class IRouter {
        <<interface>>
        +resolveOpcodeTargets(uint16_t opcode) vector~ForwardingTarget~
        +addRoute(RouteConfig route) Result
        +addPath(TopologyPath path) Result
        +findBestPath(string src, string dst) Path
        +handleTopologyChange(string nodeId, bool isAvailable)
    }

    class IProtocol {
        <<interface>>
        +encode(Message msg, vector~uint8_t~& output) Result
        +decode(span~uint8_t~ data, Message& msg, ProtocolMeta& meta) Result
        +encodeAck(uint32_t token, vector~uint8_t~& output) Result
        +getCapabilities() ProtocolCapabilities
    }

    class IDriver {
        <<interface>>
        +initialize() Result
        +send(span~uint8_t~ data) Result
        +receive(vector~uint8_t~& data, uint32_t timeoutMs) Result
        +getCapabilities() DriverCapabilities
        +shutdown() void
        +registerReceiveCallback(function~void(span~uint8_t~)~ callback)
    }

    class IDriverFactory {
        <<interface>>
        +createDriver(DriverConfig config) unique_ptr~IDriver~
        +supportsDriverType(string type, string variant) bool
    }

    class IProtocolFactory {
        <<interface>>
        +createProtocol(ProtocolConfig config) unique_ptr~IProtocol~
        +supportsProtocolType(string type) bool
    }

    %% Core message structures
    class Message {
        +uint16_t opcode
        +span~const uint8_t~ payload
    }

    class MessageContext {
        +Priority priority
        +AckPolicy ackPolicy
        +function~void(bool)~ ackCallback
        +uint32_t timeoutMs
        +bool isRetry
    }

    class ForwardingTarget {
        +string nextHop
        +string driverId
        +string protocolId
        +string finalDestination
        +vector~string~ remainingHops
        +bool isLocal
        +size_t subscriptionIndex
    }

    class ProtocolMeta {
        +bool isForwarded
        +string finalDestination
        +uint32_t token
        +uint8_t hopCount
        +bool requiresAck
        +Priority priority
    }

    %% Main managers
    class CommManager {
        -OpcodeRegistry m_opcodeRegistry
        -shared_ptr~IRouter~ m_router
        -shared_ptr~DriverManager~ m_driverManager
        -shared_ptr~ProtocolManager~ m_protocolManager
        -shared_ptr~SubscriptionManager~ m_subscriptionManager
        -shared_ptr~QoSManager~ m_qosManager
        -array~OsQueue~PriorityMessage~, 4~ m_priorityQueues
        -atomic~bool~ m_isRunning
        -thread m_processingThread
        -mutex m_statsMutex
        -Stats m_stats
        -shared_ptr~CircuitBreaker~ m_circuitBreaker

        +CommManager()
        +~CommManager()
        +publish(Message msg, MessageContext ctx) Result
        +subscribe(uint16_t opcode, function~void(Message)~ callback) SubscriptionId
        +unsubscribe(uint16_t opcode, SubscriptionId id) Result
        +unsubscribeAll(uint16_t opcode) Result
        +getStatistics() Stats
        +ingest(string_view driverId, span~uint8_t~ data) Result
        +start() Result
        +stop() Result
        +setRouter(shared_ptr~IRouter~ router)
        +setDriverManager(shared_ptr~DriverManager~ driverManager)
        +setProtocolManager(shared_ptr~ProtocolManager~ protocolManager)
        +setSubscriptionManager(shared_ptr~SubscriptionManager~ subscriptionManager)
        +setQoSManager(shared_ptr~QoSManager~ qosManager)
        +initialize() Result
        -processQueues() void
        -deliverToLocalSubscribers(Message msg) Result
        -forwardMessage(Message msg, ForwardingTarget target) Result
        -scheduleRetry(Message msg, MessageContext ctx, uint32_t delayMs) void
        -handleIncomingAck(uint32_t token, bool success) void
        -updateStatistics(StatAction action, uint64_t value) void
        -instance() CommManager&
    }

    class OpcodeRegistry {
        -unordered_map~uint16_t, OpcodeInfo~ m_opcodes
        -unordered_map~string, uint16_t~ m_opcodesByName
        -unordered_map~uint16_t, OpcodeInfo~ m_commonOpcodeCache
        -mutex m_mutex
        -unordered_map~uint16_t, uint64_t~ m_usageCount
        -function~void(OpcodeInfo)~ m_registryUpdateCallback

        +registerOpcode(OpcodeInfo info) Result
        +getOpcodeInfo(uint16_t opcode) optional~OpcodeInfo~
        +getOpcodeByName(string name) optional~uint16_t~
        +getAllOpcodes() vector~OpcodeInfo~
        +validateOpcodeAndGetInfo(uint16_t opcode, OpcodeInfo& outInfo) bool
        +deprecateOpcode(uint16_t opcode, uint16_t replacementOpcode)
        +registerOpcodeUpdateCallback(function~void(OpcodeInfo)~ callback)
        +batchRegister(vector~OpcodeInfo~ opcodes) Result
        +validateOpcodeRange(uint16_t opcode, string name) bool
        -updateUsageStatistics(uint16_t opcode) void
        -isCommonOpcode(uint16_t opcode) bool
        -isReservedName(string name) bool
        -overwriteAllowed(uint16_t opcode) bool
    }

    class RouteManager {
        -string m_nodeId
        -unordered_map~string, RouteConfig~ m_routes
        -unordered_map~string, TopologyPath~ m_paths
        -unordered_map~string, RouteGraphNode~ m_routeGraph
        -unordered_map~string, Path~ m_pathCache
        -unordered_map~uint16_t, CacheEntry~ m_targetCache
        -unordered_map~uint16_t, vector~size_t~~ m_localSubscriptions
        -unordered_map~uint16_t, set~string~~ m_remoteSubscriptions
        -unordered_set~string~ m_unavailableNodes
        -shared_mutex m_routeMutex
        -weak_ptr~DriverManager~ m_driverManager
        -uint64_t m_cacheLifetimeUs
        -size_t m_nextSubscriptionIndex

        +RouteManager(string nodeId)
        +resolveOpcodeTargets(uint16_t opcode) vector~ForwardingTarget~
        +addRoute(RouteConfig route) Result
        +addPath(TopologyPath path) Result
        +rebuildRoutingTables() void
        +registerSubscription(uint16_t opcode, string nodeId, size_t subIndex)
        +unregisterSubscription(uint16_t opcode, string nodeId, size_t subIndex)
        +findBestPath(string src, string dst) Path
        +handleTopologyChange(string nodeId, bool isAvailable)
        +setDriverManager(shared_ptr~DriverManager~ driverManager)
        +setCacheLifetime(uint64_t microseconds)
        +getRoutes() unordered_map~string, RouteConfig~
        +getPaths() unordered_map~string, TopologyPath~
        -runDijkstraAlgorithm(string src, string dst) Path
        -computeAllPairShortestPaths() void
        -validatePathContinuity(TopologyPath path) bool
        -computeRouteCost(RouteConfig route) uint32_t
        -evaluateCapabilityConstraint(DriverCapabilities caps, string constraint) bool
        -clearPathCachesForNode(string nodeId) void
        -invalidatePathCache(string src, string dst) void
        -precomputePath(TopologyPath path) void
        -isCacheExpired(CacheEntry entry) bool
        -findRouteId(string src, string dst) string
    }

    class DriverManager {
        -unordered_map~string, unique_ptr~IDriver~~ m_drivers
        -shared_ptr~IDriverFactory~ m_driverFactory
        -shared_mutex m_driverMutex
        -function~void(string_view, span~uint8_t~)~ m_receiveCallback
        -unordered_map~string, DriverStatistics~ m_driverStats
        -shared_ptr~OsTimer~ m_healthCheckTimer

        +DriverManager(shared_ptr~IDriverFactory~ factory)
        +registerDriver(string id, unique_ptr~IDriver~ driver) Result
        +getDriver(string id) IDriver*
        +sendData(string driverId, span~uint8_t~ data) Result
        +registerReceiveCallback(function~void(string_view, span~uint8_t~)~ callback)
        +createAndRegisterDriver(DriverConfig config) Result
        +removeDriver(string id) Result
        +setDriverFactory(shared_ptr~IDriverFactory~ factory)
        +getDriverStatistics(string id) optional~DriverStatistics~
        +getAllDriverIds() vector~string~
        +startHealthChecks(uint32_t intervalMs)
        +stopHealthChecks()
        -receiveHandler(string driverId, span~uint8_t~ data) void
        -performHealthCheck() void
        -updateDriverStats(string id, DriverStatAction action, uint64_t value)
    }

    class ProtocolManager {
        -unordered_map~string, unique_ptr~IProtocol~~ m_protocols
        -shared_ptr~IProtocolFactory~ m_protocolFactory
        -shared_mutex m_protocolMutex
        -unordered_map~string, ProtocolStatistics~ m_protocolStats

        +ProtocolManager(shared_ptr~IProtocolFactory~ factory)
        +getProtocol(string id) IProtocol*
        +registerProtocol(string id, unique_ptr~IProtocol~ protocol) Result
        +createAndRegisterProtocol(ProtocolConfig config) Result
        +removeProtocol(string id) Result
        +setProtocolFactory(shared_ptr~IProtocolFactory~ factory)
        +getProtocolStatistics(string id) optional~ProtocolStatistics~
        +getAllProtocolIds() vector~string~
        -updateProtocolStats(string id, ProtocolStatAction action, uint64_t value)
    }

    class ConfigManager {
        -SystemConfig m_currentConfig
        -shared_mutex m_configMutex
        -function~void(SystemConfig)~ m_configChangeCallback
        -SystemConfig m_backupConfig
        -vector~function~bool(SystemConfig)~~ m_validators
        -shared_ptr~CommManager~ m_commManager
        -string m_configFilePath

        +ConfigManager(shared_ptr~CommManager~ commManager)
        +loadConfig(string filePath) Result
        +hotSwapConfig(SystemConfig newConfig) Result
        +getCurrentConfig() SystemConfig
        +registerConfigChangeCallback(function~void(SystemConfig)~ callback)
        +registerValidator(function~bool(SystemConfig)~ validator)
        +saveConfigToFile(string filePath) Result
        +revertToLastConfig() Result
        +validateConfig(SystemConfig config) ValidationResult
        -applyConfiguration(SystemConfig config) Result
        -rollbackConfiguration() void
        -computeConfigHash(SystemConfig config) string
        -loadJsonConfig(string filePath) optional~SystemConfig~
        -saveJsonConfig(SystemConfig config, string filePath) bool
        -hasCircularDependencies(vector~TopologyPath~ paths) bool
        -findDriverById(vector~DriverConfig~ drivers, string id) iterator
        -meetsCapabilityRequirement(unordered_map~string, string~ capabilities, string requirement) bool
    }

    class QoSManager {
        -array~QoSProfile, 4~ m_profiles
        -array~atomic~uint64_t~, 4~ m_messageCounters
        -array~atomic~uint64_t~, 4~ m_resourceUsage
        -array~uint32_t, 4~ m_resourceLimits
        -shared_mutex m_qosMutex
        -shared_ptr~OsTimer~ m_monitoringTimer
        -deque~QoSSnapshot~ m_historySnapshots
        -bool m_backpressureEnabled
        -array~function~void(QoSAlert)~, 4~ m_alertCallbacks

        +QoSManager()
        +configureQoS(vector~QoSProfile~ profiles) Result
        +getSettingsForPriority(Priority priority) QoSSettings
        +allocateResources(Priority priority) Result
        +releaseResources(Priority priority) void
        +getTimeoutForPriority(Priority priority) uint32_t
        +getRetryDelayForPriority(Priority priority) uint32_t
        +getStatistics() QoSStatistics
        +registerAlertCallback(Priority priority, function~void(QoSAlert)~ callback)
        +setResourceLimits(array~uint32_t, 4~ limits)
        +enableBackpressure(bool enable)
        +startMonitoring(uint32_t intervalMs)
        +stopMonitoring()
        -takeSnapshot() void
        -checkThresholds() void
        -calculatePriorityWeight(Priority priority) uint32_t
    }

    class SubscriptionManager {
        -unordered_map~uint16_t, vector~SubscriptionEntry~~ m_subscriptions
        -shared_mutex m_subscriptionMutex
        -atomic~uint64_t~ m_nextSubscriptionId
        -weak_ptr~OpcodeRegistry~ m_opcodeRegistry
        -weak_ptr~RouteManager~ m_routeManager
        -unordered_map~uint16_t, unordered_set~uint64_t~~ m_activeSubscriptions

        +SubscriptionManager()
        +subscribe(uint16_t opcode, function~void(Message)~ callback) SubscriptionId
        +unsubscribe(uint16_t opcode, SubscriptionId id) bool
        +getSubscribers(uint16_t opcode) vector~SubscriptionEntry~
        +deliverMessage(Message msg)
        +getSubscriptionCount() size_t
        +getActiveOpcodes() vector~uint16_t~
        +setOpcodeRegistry(shared_ptr~OpcodeRegistry~ registry)
        +setRouteManager(shared_ptr~RouteManager~ routeManager)
        -notifyRouteManager(uint16_t opcode, bool isSubscribing, SubscriptionId id)
    }

    %% Protocol implementations
    class CustomAckProtocol {
        -uint32_t m_ackTimeout
        -uint32_t m_maxRetries
        -float m_backoffMultiplier
        -bool m_enableCrc

        +CustomAckProtocol(ProtocolConfig config)
        +encode(Message msg, vector~uint8_t~& output) Result
        +decode(span~uint8_t~ data, Message& msg, ProtocolMeta& meta) Result
        +encodeAck(uint32_t token, vector~uint8_t~& output) Result
        +getCapabilities() ProtocolCapabilities
        -encodeForward(Message msg, ForwardingMeta meta, vector~uint8_t~& output) Result
        -validateCrc(span~uint8_t~ data, uint16_t expectedCrc) bool
        -calculateCrc(span~uint8_t~ data) uint16_t
        -generateToken() uint32_t
    }

    class CustomLiteProtocol {
        -bool m_enableCrc
        -bool m_headerCompression

        +CustomLiteProtocol(ProtocolConfig config)
        +encode(Message msg, vector~uint8_t~& output) Result
        +decode(span~uint8_t~ data, Message& msg, ProtocolMeta& meta) Result
        +encodeAck(uint32_t token, vector~uint8_t~& output) Result
        +getCapabilities() ProtocolCapabilities
        -compressHeader(Message msg) vector~uint8_t~
        -decompressHeader(span~uint8_t~ data) pair~Message, bool~
        -calculateCrc8(span~uint8_t~ data) uint8_t
    }

    class SecureAesProtocol {
        -uint32_t m_ackTimeout
        -uint32_t m_maxRetries
        -string m_encryptionKeyId
        -bool m_authenticationEnabled
        -string m_authenticationMethod
        -unique_ptr~SecurityProvider~ m_securityProvider

        +SecureAesProtocol(ProtocolConfig config)
        +encode(Message msg, vector~uint8_t~& output) Result
        +decode(span~uint8_t~ data, Message& msg, ProtocolMeta& meta) Result
        +encodeAck(uint32_t token, vector~uint8_t~& output) Result
        +getCapabilities() ProtocolCapabilities
        -encryptPayload(span~uint8_t~ data, vector~uint8_t~& output) Result
        -decryptPayload(span~uint8_t~ data, vector~uint8_t~& output) Result
        -calculateHmac(span~uint8_t~ data) vector~uint8_t~
        -verifyHmac(span~uint8_t~ data, span~uint8_t~ expectedHmac) bool
    }

    %% Driver implementations
    class IpcDriver {
        -string m_interface
        -uint32_t m_queueSize
        -uint32_t m_sharedMemSize
        -function~void(span~uint8_t~)~ m_receiveCallback
        -void* m_sharedMemoryRegion
        -atomic~bool~ m_isRunning
        -thread m_receiveThread
        -OsMutex m_sendMutex

        +IpcDriver(DriverConfig config)
        +initialize() Result
        +send(span~uint8_t~ data) Result
        +receive(vector~uint8_t~& data, uint32_t timeoutMs) Result
        +getCapabilities() DriverCapabilities
        +shutdown() void
        +registerReceiveCallback(function~void(span~uint8_t~)~ callback)
        -handleInterrupt() void
        -receiveLoop() void
        -mapSharedMemory() bool
        -unmapSharedMemory() void
    }

    class I2CDriver {
        -string m_interface
        -uint32_t m_speed
        -uint8_t m_address
        -bool m_isDma
        -function~void(span~uint8_t~)~ m_receiveCallback
        -void* m_i2cHandle
        -atomic~bool~ m_isRunning
        -OsMutex m_sendMutex
        -OsDmaController m_dmaController

        +I2CDriver(DriverConfig config)
        +initialize() Result
        +send(span~uint8_t~ data) Result
        +receive(vector~uint8_t~& data, uint32_t timeoutMs) Result
        +getCapabilities() DriverCapabilities
        +shutdown() void
        +registerReceiveCallback(function~void(span~uint8_t~)~ callback)
        -handleInterrupt() void
        -configureDma() bool
        -waitForCompletion(uint32_t timeoutMs) bool
    }

    class SpiDriver {
        -string m_interface
        -uint32_t m_speed
        -uint8_t m_mode
        -bool m_isDma
        -function~void(span~uint8_t~)~ m_receiveCallback
        -void* m_spiHandle
        -atomic~bool~ m_isRunning
        -OsMutex m_sendMutex
        -OsDmaController m_dmaController

        +SpiDriver(DriverConfig config)
        +initialize() Result
        +send(span~uint8_t~ data) Result
        +receive(vector~uint8_t~& data, uint32_t timeoutMs) Result
        +getCapabilities() DriverCapabilities
        +shutdown() void
        +registerReceiveCallback(function~void(span~uint8_t~)~ callback)
        -handleInterrupt() void
        -configureDma() bool
        -waitForCompletion(uint32_t timeoutMs) bool
    }

    %% Factory implementations
    class StandardDriverFactory {
        -unordered_map~string, function~unique_ptr~IDriver~(DriverConfig)~~ m_creators

        +StandardDriverFactory()
        +createDriver(DriverConfig config) unique_ptr~IDriver~
        +supportsDriverType(string type, string variant) bool
        +registerCreator(string type, string variant, function~unique_ptr~IDriver~(DriverConfig)~)
        -createIpcDriver(DriverConfig config) unique_ptr~IDriver~
        -createI2CDriver(DriverConfig config) unique_ptr~IDriver~
        -createSpiDriver(DriverConfig config) unique_ptr~IDriver~
    }

    class StandardProtocolFactory {
        -unordered_map~string, function~unique_ptr~IProtocol~(ProtocolConfig)~~ m_creators

        +StandardProtocolFactory()
        +createProtocol(ProtocolConfig config) unique_ptr~IProtocol~
        +supportsProtocolType(string type) bool
        +registerCreator(string type, function~unique_ptr~IProtocol~(ProtocolConfig)~)
        -createCustomAckProtocol(ProtocolConfig config) unique_ptr~IProtocol~
        -createCustomLiteProtocol(ProtocolConfig config) unique_ptr~IProtocol~
        -createSecureAesProtocol(ProtocolConfig config) unique_ptr~IProtocol~
    }

    %% OS abstraction
    class OsMutex {
        -void* m_handle

        +OsMutex()
        +~OsMutex()
        +lock() void
        +unlock() void
        +tryLock() bool
    }

    class OsQueue~T~ {
        -void* m_handle
        -size_t m_capacity

        +OsQueue(size_t capacity)
        +~OsQueue()
        +push(T item, uint32_t timeoutMs) bool
        +pop(T& item, uint32_t timeoutMs) bool
        +size() size_t
        +empty() bool
        +clear() void
    }

    class OsTimer {
        -void* m_handle
        -uint32_t m_intervalMs
        -bool m_isPeriodic
        -function~void()~ m_callback
        -atomic~bool~ m_isRunning

        +OsTimer(uint32_t intervalMs, bool isPeriodic, function~void()~ callback)
        +~OsTimer()
        +start() void
        +stop() void
        +setPeriod(uint32_t intervalMs) void
        +isRunning() bool
    }

    class OsClock {
        +nowMicros()$ uint64_t
        +nowMillis()$ uint64_t
        +sleep(uint32_t milliseconds)$ void
    }

    %% Utility classes
    class CircuitBreaker {
        -uint32_t m_failureThreshold
        -uint32_t m_resetTimeoutMs
        -atomic~uint32_t~ m_failureCount
        -atomic~uint64_t~ m_lastFailureTime
        -atomic~bool~ m_isOpen
        -function~void(bool)~ m_stateChangeCallback

        +CircuitBreaker(uint32_t failureThreshold, uint32_t resetTimeoutMs)
        +recordSuccess() void
        +recordFailure() bool
        +isOpen() bool
        +reset() void
        +setStateChangeCallback(function~void(bool)~ callback)
    }

    class DiagnosticsCollector {
        -OsQueue~DiagnosticEvent~ m_eventQueue
        -unordered_map~string, PerformanceMetrics~ m_metrics
        -thread m_processingThread
        -atomic~bool~ m_isRunning
        -shared_mutex m_metricsMutex
        -vector~function~void(DiagnosticAlert)~~ m_alertCallbacks

        +DiagnosticsCollector()
        +~DiagnosticsCollector()
        +recordEvent(DiagnosticEvent event) void
        +getPerformanceMetrics() PerformanceMetrics
        +getMetricsForCategory(string category) optional~PerformanceMetrics~
        +startTracing(uint16_t opcode) TraceId
        +stopTracing(TraceId id) TraceResults
        +start() void
        +stop() void
        +registerAlertCallback(function~void(DiagnosticAlert)~ callback)
        -processEvents() void
        -updateMetrics(DiagnosticEvent event) void
        -checkThresholds() void
    }

    %% Relationships
    IProtocol <|-- CustomAckProtocol
    IProtocol <|-- CustomLiteProtocol
    IProtocol <|-- SecureAesProtocol

    IDriver <|-- IpcDriver
    IDriver <|-- I2CDriver
    IDriver <|-- SpiDriver

    IDriverFactory <|-- StandardDriverFactory
    IProtocolFactory <|-- StandardProtocolFactory

    IRouter <|-- RouteManager

    CommManager *-- OpcodeRegistry
    CommManager o-- IRouter
    CommManager o-- DriverManager
    CommManager o-- ProtocolManager
    CommManager o-- SubscriptionManager
    CommManager o-- QoSManager
    CommManager *-- CircuitBreaker
    CommManager o-- OsQueue
    CommManager o-- OsTimer

    DriverManager *-- IDriver
    DriverManager o-- IDriverFactory
    DriverManager o-- OsTimer

    ProtocolManager *-- IProtocol
    ProtocolManager o-- IProtocolFactory

    ConfigManager o-- CommManager

    RouteManager o-- DriverManager
    SubscriptionManager o-- OpcodeRegistry
    SubscriptionManager o-- RouteManager

    OsTimer o-- OsClock
```

## Core Components and Responsibilities

### 1. CommManager
- **Core Functionality**: Central hub for messaging operations
- **Responsibilities**:
  - Publish/subscribe API for applications
  - Message queuing and prioritization
  - Routing via IRouter
  - Protocol selection and encoding
  - Driver selection for transmission
  - Statistics and monitoring

### 2. OpcodeRegistry
- **Core Functionality**: OPCODE definition management
- **Responsibilities**:
  - OPCODE registration and validation
  - Metadata management
  - Version tracking and deprecation
  - High-performance lookup optimization

### 3. RouteManager
- **Core Functionality**: Message routing across distributed system
- **Responsibilities**:
  - Path finding using Dijkstra's algorithm
  - Multi-hop route resolution
  - Topology management
  - Subscription-based routing
  - Path caching for performance

### 4. DriverManager
- **Core Functionality**: Hardware communication abstraction
- **Responsibilities**:
  - Driver lifecycle management
  - Send/receive operations
  - Driver factory integration
  - Health monitoring
  - Statistics collection

### 5. ProtocolManager
- **Core Functionality**: Protocol selection and message encoding
- **Responsibilities**:
  - Protocol lifecycle management
  - Protocol factory integration
  - Statistics collection
  - Protocol selection based on message properties

### 6. ConfigManager
- **Core Functionality**: Configuration loading and hot-swapping
- **Responsibilities**:
  - Configuration validation
  - Atomic configuration updates
  - Rollback capability
  - Configuration persistence
  - Hash verification

### 7. QoSManager
- **Core Functionality**: Quality of service enforcement
- **Responsibilities**:
  - Priority-based resource allocation
  - Timeout management
  - Backpressure handling
  - Performance monitoring
  - Threshold alerts

### 8. SubscriptionManager
- **Core Functionality**: Message subscription handling
- **Responsibilities**:
  - Subscription registration
  - Message delivery to subscribers
  - Subscription lifecycle management
  - Integration with routing system

## Detailed Component Implementations

### Protocol Implementations

```cpp
class CustomAckProtocol : public IProtocol {
public:
    CustomAckProtocol(const ProtocolConfig& config) {
        m_ackTimeout = std::stoul(config.settings.at("ackTimeout"));
        m_maxRetries = std::stoul(config.settings.at("maxRetries"));
        m_backoffMultiplier = std::stof(config.settings.at("backoffMultiplier"));
        m_enableCrc = config.settings.at("enableCRC") == "true";
    }

    Result encode(const Message& msg, std::vector<uint8_t>& output) override {
        output.clear();

        // Allocate space for header + payload + CRC
        output.reserve(kHeaderSize + msg.payload.size() + (m_enableCrc ? sizeof(uint16_t) : 0));

        // Header: [OPCODE:2][LENGTH:2][FLAGS:1][TOKEN:4][SEQ:1]
        uint8_t flags = 0;
        uint32_t token = generateToken();

        // Write header
        output.push_back(static_cast<uint8_t>(msg.opcode & 0xFF));
        output.push_back(static_cast<uint8_t>((msg.opcode >> 8) & 0xFF));

        uint16_t length = static_cast<uint16_t>(msg.payload.size());
        output.push_back(static_cast<uint8_t>(length & 0xFF));
        output.push_back(static_cast<uint8_t>((length >> 8) & 0xFF));

        output.push_back(flags);

        // Token (for acknowledgment)
        for (int i = 0; i < 4; ++i) {
            output.push_back(static_cast<uint8_t>((token >> (i * 8)) & 0xFF));
        }

        // Sequence number (for fragmentation)
        output.push_back(0); // Not fragmented

        // Copy payload
        output.insert(output.end(), msg.payload.begin(), msg.payload.end());

        // Add CRC if enabled
        if (m_enableCrc) {
            uint16_t crc = calculateCrc(std::span<const uint8_t>(output.data(), output.size()));
            output.push_back(static_cast<uint8_t>(crc & 0xFF));
            output.push_back(static_cast<uint8_t>((crc >> 8) & 0xFF));
        }

        return Result::Success;
    }

    Result decode(std::span<const uint8_t> data, Message& msg, ProtocolMeta& meta) override {
        // Check minimum size
        if (data.size() < kHeaderSize) {
            return Result::InvalidFormat;
        }

        // Verify CRC if enabled
        if (m_enableCrc) {
            size_t dataSize = data.size() - sizeof(uint16_t);
            uint16_t expectedCrc = static_cast<uint16_t>(data[dataSize]) |
                                  (static_cast<uint16_t>(data[dataSize + 1]) << 8);

            if (!validateCrc(data.subspan(0, dataSize), expectedCrc)) {
                return Result::CrcError;
            }
        }

        // Parse header
        msg.opcode = static_cast<uint16_t>(data[0]) | (static_cast<uint16_t>(data[1]) << 8);

        uint16_t length = static_cast<uint16_t>(data[2]) | (static_cast<uint16_t>(data[3]) << 8);
        uint8_t flags = data[4];

        // Extract token
        meta.token = 0;
        for (int i = 0; i < 4; ++i) {
            meta.token |= static_cast<uint32_t>(data[5 + i]) << (i * 8);
        }

        // Check if this is an ACK message
        if (flags & kAckFlag) {
            meta.isAck = true;
            return Result::Success;
        }

        // Extract payload
        if (data.size() < kHeaderSize + length + (m_enableCrc ? sizeof(uint16_t) : 0)) {
            return Result::InvalidFormat;
        }

        msg.payload = data.subspan(kHeaderSize, length);

        // Fill other metadata
        meta.isForwarded = (flags & kForwardFlag) != 0;
        meta.requiresAck = (flags & kRequireAckFlag) != 0;
        meta.hopCount = data[9]; // Sequence byte repurposed for hop count in forwarded messages

        return Result::Success;
    }

    Result encodeAck(uint32_t token, std::vector<uint8_t>& output) override {
        output.clear();
        output.reserve(kAckSize + (m_enableCrc ? sizeof(uint16_t) : 0));

        // ACK header: [0:2][0:2][FLAGS:1][TOKEN:4]
        uint8_t flags = kAckFlag;

        // Write header with zeroed opcode and length
        output.push_back(0);
        output.push_back(0);
        output.push_back(0);
        output.push_back(0);

        output.push_back(flags);

        // Token (same as original message)
        for (int i = 0; i < 4; ++i) {
            output.push_back(static_cast<uint8_t>((token >> (i * 8)) & 0xFF));
        }

        // Add CRC if enabled
        if (m_enableCrc) {
            uint16_t crc = calculateCrc(std::span<const uint8_t>(output.data(), output.size()));
            output.push_back(static_cast<uint8_t>(crc & 0xFF));
            output.push_back(static_cast<uint8_t>((crc >> 8) & 0xFF));
        }

        return Result::Success;
    }

    ProtocolCapabilities getCapabilities() override {
        ProtocolCapabilities caps;
        caps.supportsAck = true;
        caps.supportsFragmentation = false;
        caps.supportsSecurity = false;
        caps.maxMessageSize = 65535; // Based on length field size
        caps.overhead = kHeaderSize + (m_enableCrc ? sizeof(uint16_t) : 0);
        return caps;
    }

private:
    static constexpr size_t kHeaderSize = 10; // OPCODE(2) + LENGTH(2) + FLAGS(1) + TOKEN(4) + SEQ(1)
    static constexpr size_t kAckSize = 9;     // ZEROS(4) + FLAGS(1) + TOKEN(4)
    static constexpr uint8_t kAckFlag = 0x01;
    static constexpr uint8_t kForwardFlag = 0x02;
    static constexpr uint8_t kRequireAckFlag = 0x04;

    uint32_t m_ackTimeout;
    uint32_t m_maxRetries;
    float m_backoffMultiplier;
    bool m_enableCrc;

    Result encodeForward(const Message& msg, const ForwardingMeta& meta, std::vector<uint8_t>& output) {
        output.clear();

        // Allocate space for header + payload + CRC
        output.reserve(kHeaderSize + msg.payload.size() + (m_enableCrc ? sizeof(uint16_t) : 0));

        // Header: [OPCODE:2][LENGTH:2][FLAGS:1][TOKEN:4][HOPCOUNT:1]
        uint8_t flags = kForwardFlag;
        if (meta.requiresAck) {
            flags |= kRequireAckFlag;
        }

        // Write header
        output.push_back(static_cast<uint8_t>(msg.opcode & 0xFF));
        output.push_back(static_cast<uint8_t>((msg.opcode >> 8) & 0xFF));

        uint16_t length = static_cast<uint16_t>(msg.payload.size());
        output.push_back(static_cast<uint8_t>(length & 0xFF));
        output.push_back(static_cast<uint8_t>((length >> 8) & 0xFF));

        output.push_back(flags);

        // Token (preserved from original message)
        for (int i = 0; i < 4; ++i) {
            output.push_back(static_cast<uint8_t>((meta.token >> (i * 8)) & 0xFF));
        }

        // Hop count
        output.push_back(meta.hopCount + 1);

        // Copy payload
        output.insert(output.end(), msg.payload.begin(), msg.payload.end());

        // Add CRC if enabled
        if (m_enableCrc) {
            uint16_t crc = calculateCrc(std::span<const uint8_t>(output.data(), output.size()));
            output.push_back(static_cast<uint8_t>(crc & 0xFF));
            output.push_back(static_cast<uint8_t>((crc >> 8) & 0xFF));
        }

        return Result::Success;
    }

    bool validateCrc(std::span<const uint8_t> data, uint16_t expectedCrc) {
        return calculateCrc(data) == expectedCrc;
    }

    uint16_t calculateCrc(std::span<const uint8_t> data) {
        // CRC-16 CCITT implementation
        uint16_t crc = 0xFFFF;
        for (uint8_t byte : data) {
            crc ^= static_cast<uint16_t>(byte) << 8;
            for (int i = 0; i < 8; ++i) {
                if (crc & 0x8000) {
                    crc = (crc << 1) ^ 0x1021;
                } else {
                    crc <<= 1;
                }
            }
        }
        return crc;
    }

    uint32_t generateToken() {
        static std::mt19937 rng(std::random_device{}());
        static std::uniform_int_distribution<uint32_t> dist(1, std::numeric_limits<uint32_t>::max());
        return dist(rng);
    }
};
```

### Driver Implementation

```cpp
class I2CDriver : public IDriver {
public:
    I2CDriver(const DriverConfig& config)
        : m_interface(config.settings.at("interface")),
          m_speed(std::stoul(config.settings.at("speed"))),
          m_address(static_cast<uint8_t>(std::stoul(config.settings.at("address"), nullptr, 16))),
          m_isDma(config.settings.at("dma") == "true"),
          m_i2cHandle(nullptr),
          m_isRunning(false) {
    }

    ~I2CDriver() {
        shutdown();
    }

    Result initialize() override {
        if (m_i2cHandle != nullptr) {
            // Already initialized
            return Result::Success;
        }

        try {
            // Platform-specific I2C initialization
            #if defined(STM32_PLATFORM)
                I2C_HandleTypeDef* handle = new I2C_HandleTypeDef();
                handle->Instance = getI2CInstance(m_interface);

                I2C_InitTypeDef init = {};
                init.ClockSpeed = m_speed;
                init.DutyCycle = I2C_DUTYCYCLE_2;
                init.OwnAddress1 = 0;
                init.AddressingMode = I2C_ADDRESSINGMODE_7BIT;
                init.DualAddressMode = I2C_DUALADDRESS_DISABLE;
                init.OwnAddress2 = 0;
                init.GeneralCallMode = I2C_GENERALCALL_DISABLE;
                init.NoStretchMode = I2C_NOSTRETCH_DISABLE;

                handle->Init = init;

                if (HAL_I2C_Init(handle) != HAL_OK) {
                    delete handle;
                    return Result::HardwareError;
                }

                m_i2cHandle = handle;

                // Setup DMA if enabled
                if (m_isDma && !configureDma()) {
                    HAL_I2C_DeInit(handle);
                    delete handle;
                    m_i2cHandle = nullptr;
                    return Result::HardwareError;
                }

                // Setup interrupt
                if (m_receiveCallback) {
                    HAL_I2C_RegisterCallback(handle, HAL_I2C_RX_COMPLETE_CB_ID,
                                            [](I2C_HandleTypeDef* h) {
                                                // Get driver instance from handle and call its handler
                                                I2CDriver* driver = getDriverFromHandle(h);
                                                if (driver) {
                                                    driver->handleInterrupt();
                                                }
                                            });
                }
            #elif defined(LINUX_PLATFORM)
                // Linux I2C implementation using i2c-dev
                int fd = open(("/dev/i2c-" + m_interface).c_str(), O_RDWR);
                if (fd < 0) {
                    return Result::HardwareError;
                }

                // Set I2C slave address
                if (ioctl(fd, I2C_SLAVE, m_address) < 0) {
                    close(fd);
                    return Result::HardwareError;
                }

                // Store handle as integer pointer
                m_i2cHandle = reinterpret_cast<void*>(static_cast<intptr_t>(fd));

                // Start receive thread for Linux implementation
                m_isRunning = true;
                std::thread receiveThread([this]() {
                    receiveLoop();
                });
                receiveThread.detach();
            #else
                // Default platform not supported
                return Result::NotSupported;
            #endif

            return Result::Success;
        } catch (const std::exception& e) {
            // Log error
            return Result::HardwareError;
        }
    }

    Result send(std::span<const uint8_t> data) noexcept override {
        if (m_i2cHandle == nullptr) {
            return Result::NotInitialized;
        }

        // Check for max frame size
        if (data.size() > getCapabilities().maxFrameSize) {
            return Result::InvalidSize;
        }

        std::lock_guard<OsMutex> lock(m_sendMutex);

        try {
            #if defined(STM32_PLATFORM)
                I2C_HandleTypeDef* handle = static_cast<I2C_HandleTypeDef*>(m_i2cHandle);

                HAL_StatusTypeDef status;
                if (m_isDma) {
                    status = HAL_I2C_Master_Transmit_DMA(handle, m_address,
                                                       const_cast<uint8_t*>(data.data()),
                                                       data.size());

                    if (status == HAL_OK) {
                        // Wait for completion with timeout
                        if (!waitForCompletion(5000)) {
                            return Result::Timeout;
                        }
                    }
                } else {
                    status = HAL_I2C_Master_Transmit(handle, m_address,
                                                   const_cast<uint8_t*>(data.data()),
                                                   data.size(), 5000);
                }

                if (status != HAL_OK) {
                    return Result::HardwareError;
                }
            #elif defined(LINUX_PLATFORM)
                int fd = static_cast<int>(reinterpret_cast<intptr_t>(m_i2cHandle));

                // For Linux, write directly to file descriptor
                ssize_t written = write(fd, data.data(), data.size());
                if (written != static_cast<ssize_t>(data.size())) {
                    return Result::HardwareError;
                }
            #else
                return Result::NotSupported;
            #endif

            return Result::Success;
        } catch (const std::exception& e) {
            // Log error
            return Result::HardwareError;
        }
    }

    Result receive(std::vector<uint8_t>& data, uint32_t timeoutMs = 0) noexcept override {
        // This implementation uses callbacks for STM32 and a polling thread for Linux
        // So this method is only used for explicit polling, not the main receive path
        if (m_i2cHandle == nullptr) {
            return Result::NotInitialized;
        }

        try {
            #if defined(STM32_PLATFORM)
                I2C_HandleTypeDef* handle = static_cast<I2C_HandleTypeDef*>(m_i2cHandle);

                // Allocate receive buffer
                data.resize(getCapabilities().maxFrameSize);

                // Read from I2C with timeout
                HAL_StatusTypeDef status = HAL_I2C_Master_Receive(
                    handle, m_address, data.data(), data.size(), timeoutMs);

                if (status == HAL_TIMEOUT) {
                    return Result::Timeout;
                } else if (status != HAL_OK) {
                    return Result::HardwareError;
                }

                // Resize based on actual data received (platform specific)
                size_t actualSize = getReceivedDataSize(handle);
                data.resize(actualSize);
            #elif defined(LINUX_PLATFORM)
                int fd = static_cast<int>(reinterpret_cast<intptr_t>(m_i2cHandle));

                // Set up poll with timeout
                struct pollfd pfd;
                pfd.fd = fd;
                pfd.events = POLLIN;

                int pollResult = poll(&pfd, 1, timeoutMs);
                if (pollResult == 0) {
                    return Result::Timeout;
                } else if (pollResult < 0) {
                    return Result::HardwareError;
                }

                // Read available data
                data.resize(getCapabilities().maxFrameSize);
                ssize_t bytesRead = read(fd, data.data(), data.size());

                if (bytesRead < 0) {
                    return Result::HardwareError;
                }

                data.resize(bytesRead);
            #else
                return Result::NotSupported;
            #endif

            return Result::Success;
        } catch (const std::exception& e) {
            // Log error
            return Result::HardwareError;
        }
    }

    DriverCapabilities getCapabilities() const override {
        DriverCapabilities caps;
        caps.maxFrameSize = 256;  // Typical I2C buffer size
        caps.supportsDMA = m_isDma;
        caps.supportsAsync = false;
        caps.latencyUs = 200;  // Approximate latency for I2C
        return caps;
    }

    void shutdown() override {
        if (m_i2cHandle == nullptr) {
            return;
        }

        m_isRunning = false;

        try {
            #if defined(STM32_PLATFORM)
                I2C_HandleTypeDef* handle = static_cast<I2C_HandleTypeDef*>(m_i2cHandle);
                HAL_I2C_DeInit(handle);
                delete handle;
            #elif defined(LINUX_PLATFORM)
                int fd = static_cast<int>(reinterpret_cast<intptr_t>(m_i2cHandle));
                close(fd);
            #endif

             m_i2cHandle = nullptr;
        } catch (const std::exception& e) {
            // Log error
        }
    }

    void registerReceiveCallback(std::function<void(std::span<uint8_t>)> callback) override {
        m_receiveCallback = std::move(callback);
    }

private:
    std::string m_interface;
    uint32_t m_speed;
    uint8_t m_address;
    bool m_isDma;
    std::function<void(std::span<uint8_t>)> m_receiveCallback;
    void* m_i2cHandle;
    std::atomic<bool> m_isRunning;
    OsMutex m_sendMutex;
    OsDmaController m_dmaController;

    void handleInterrupt() {
        if (!m_receiveCallback) {
            return;
        }

        #if defined(STM32_PLATFORM)
            I2C_HandleTypeDef* handle = static_cast<I2C_HandleTypeDef*>(m_i2cHandle);

            // Get data from DMA buffer
            size_t dataSize = getReceivedDataSize(handle);
            if (dataSize > 0) {
                std::vector<uint8_t> data(dataSize);
                memcpy(data.data(), getDmaBuffer(), dataSize);

                // Call the callback
                m_receiveCallback(std::span<uint8_t>(data));
            }
        #endif
    }

    bool configureDma() {
        #if defined(STM32_PLATFORM)
            I2C_HandleTypeDef* handle = static_cast<I2C_HandleTypeDef*>(m_i2cHandle);

            // Configure DMA for I2C
            if (!m_dmaController.initialize(handle->Instance)) {
                return false;
            }

            // Set up DMA channels for TX and RX
            if (!m_dmaController.configureChannel(DmaDirection::Tx, m_interface + "_TX")) {
                return false;
            }

            if (!m_dmaController.configureChannel(DmaDirection::Rx, m_interface + "_RX")) {
                return false;
            }

            return true;
        #else
            return false;
        #endif
    }

    bool waitForCompletion(uint32_t timeoutMs) {
        #if defined(STM32_PLATFORM)
            I2C_HandleTypeDef* handle = static_cast<I2C_HandleTypeDef*>(m_i2cHandle);

            uint32_t startTime = OsClock::nowMillis();
            while (HAL_I2C_GetState(handle) != HAL_I2C_STATE_READY) {
                if (OsClock::nowMillis() - startTime > timeoutMs) {
                    return false;
                }
                OsClock::sleep(1);
            }
            return true;
        #else
            // Non-STM32 platforms may implement differently
            OsClock::sleep(timeoutMs);
            return true;
        #endif
    }

    void receiveLoop() {
        #if defined(LINUX_PLATFORM)
            int fd = static_cast<int>(reinterpret_cast<intptr_t>(m_i2cHandle));

            std::vector<uint8_t> buffer(getCapabilities().maxFrameSize);

            while (m_isRunning) {
                // Poll with short timeout to allow for shutdown
                struct pollfd pfd;
                pfd.fd = fd;
                pfd.events = POLLIN;

                int pollResult = poll(&pfd, 1, 100);
                if (pollResult > 0 && (pfd.revents & POLLIN)) {
                    ssize_t bytesRead = read(fd, buffer.data(), buffer.size());

                    if (bytesRead > 0 && m_receiveCallback) {
                        m_receiveCallback(std::span<uint8_t>(buffer.data(), bytesRead));
                    }
                }
            }
        #endif
    }

    #if defined(STM32_PLATFORM)
    static I2C_TypeDef* getI2CInstance(const std::string& interface) {
        if (interface == "I2C1") return I2C1;
        else if (interface == "I2C2") return I2C2;
        else if (interface == "I2C3") return I2C3;
        else throw std::runtime_error("Invalid I2C interface: " + interface);
    }

    static size_t getReceivedDataSize(I2C_HandleTypeDef* handle) {
        // Implementation depends on STM32 HAL specifics
        return handle->RxXferSize - handle->RxXferCount;
    }

    static uint8_t* getDmaBuffer() {
        // Implementation depends on DMA buffer management
        extern uint8_t g_i2cDmaBuffer[256];
        return g_i2cDmaBuffer;
    }

    static I2CDriver* getDriverFromHandle(I2C_HandleTypeDef* handle) {
        // Implementation to map HAL handle back to driver instance
        extern std::unordered_map<I2C_HandleTypeDef*, I2CDriver*> g_i2cDriverMap;
        auto it = g_i2cDriverMap.find(handle);
        return (it != g_i2cDriverMap.end()) ? it->second : nullptr;
    }
    #endif
};
```

## CommManager Implementation

The CommManager is the central hub of the messaging system, handling publishing, subscription, and message routing:

```cpp name=comm_manager.hpp
class CommManager {
public:
    // Singleton access
    static CommManager& getInstance() {
        static CommManager instance;
        return instance;
    }

    // Initialize the system with dependencies
    Result initialize() {
        if (m_isRunning) {
            return Result::AlreadyInitialized;
        }

        // Validate dependencies
        if (!m_opcodeRegistry || !m_router || !m_driverManager ||
            !m_protocolManager || !m_subscriptionManager || !m_qosManager) {
            return Result::MissingDependency;
        }

        // Initialize circuit breaker for system protection
        m_circuitBreaker = std::make_shared<CircuitBreaker>(10, 5000);
        m_circuitBreaker->setStateChangeCallback([this](bool isOpen) {
            if (isOpen) {
                // Log circuit breaker trip
            } else {
                // Log circuit breaker reset
            }
        });

        // Start processing thread
        m_isRunning = true;
        m_processingThread = std::thread(&CommManager::processQueues, this);

        return Result::Success;
    }

    // Stop the messaging system
    void stop() {
        if (!m_isRunning) {
            return;
        }

        m_isRunning = false;

        // Wait for processing thread to terminate
        if (m_processingThread.joinable()) {
            m_processingThread.join();
        }
    }

    // Publish a message with optional context
    Result publish(const Message& msg, const MessageContext& ctx = {}) noexcept {
        // Check if circuit breaker is open (system protection)
        if (m_circuitBreaker->isOpen()) {
            return Result::CircuitBreakerOpen;
        }

        // 1. Validate OPCODE
        OpcodeInfo opcodeInfo;
        if (!m_opcodeRegistry->validateOpcodeAndGetInfo(msg.opcode, opcodeInfo)) {
            m_circuitBreaker->recordFailure();
            return Result::InvalidOpcode;
        }

        // Check payload size
        if (msg.payload.size() > opcodeInfo.maxPayloadSize) {
            m_circuitBreaker->recordFailure();
            return Result::PayloadTooLarge;
        }

        // 2. Determine actual context to use (apply defaults from OPCODE if needed)
        MessageContext effectiveCtx = ctx;
        if (effectiveCtx.priority == Priority::DEFAULT) {
            effectiveCtx.priority = opcodeInfo.defaultPriority;
        }
        if (effectiveCtx.ackPolicy == AckPolicy::DEFAULT) {
            effectiveCtx.ackPolicy = opcodeInfo.defaultAckPolicy;
        }

        // 3. Allocate QoS resources
        if (m_qosManager->allocateResources(effectiveCtx.priority) != Result::Success) {
            return Result::ResourceAllocationFailed;
        }

        // Create a copy of the message with its own memory
        PriorityMessage priorityMsg;
        priorityMsg.opcode = msg.opcode;
        priorityMsg.payload.assign(msg.payload.begin(), msg.payload.end());
        priorityMsg.context = effectiveCtx;
        priorityMsg.timestamp = OsClock::nowMicros();

        // 4. Queue the message according to priority
        int priorityIndex = static_cast<int>(effectiveCtx.priority);
        bool queued = m_priorityQueues[priorityIndex].push(priorityMsg, 100);

        if (!queued) {
            // Release allocated resources
            m_qosManager->releaseResources(effectiveCtx.priority);
            return Result::QueueFull;
        }

        // Update statistics
        updateStatistics(StatAction::MessagePublished);

        m_circuitBreaker->recordSuccess();
        return Result::Success;
    }

    // Subscribe to messages with a specific OPCODE
    SubscriptionId subscribe(uint16_t opcode, std::function<void(const Message&)> callback) {
        // Validate OPCODE
        OpcodeInfo opcodeInfo;
        if (!m_opcodeRegistry->validateOpcodeAndGetInfo(opcode, opcodeInfo)) {
            return INVALID_SUBSCRIPTION_ID;
        }

        // Register subscription
        return m_subscriptionManager->subscribe(opcode, std::move(callback));
    }

    // Unsubscribe from messages
    Result unsubscribe(uint16_t opcode, SubscriptionId id) {
        if (m_subscriptionManager->unsubscribe(opcode, id)) {
            return Result::Success;
        } else {
            return Result::SubscriptionNotFound;
        }
    }

    // Get current statistics
    Stats getStatistics() const {
        std::lock_guard<std::mutex> lock(m_statsMutex);
        return m_stats;
    }

    // Ingest a message from a driver
    Result ingest(std::string_view driverId, std::span<const uint8_t> data) noexcept {
        try {
            // Check if circuit breaker is open
            if (m_circuitBreaker->isOpen()) {
                return Result::CircuitBreakerOpen;
            }

            // Find the appropriate protocol based on driver
            std::string protocolId = getProtocolForDriver(std::string(driverId));
            if (protocolId.empty()) {
                m_circuitBreaker->recordFailure();
                return Result::ProtocolNotFound;
            }

            // Get protocol
            IProtocol* protocol = m_protocolManager->getProtocol(protocolId);
            if (!protocol) {
                m_circuitBreaker->recordFailure();
                return Result::ProtocolNotFound;
            }

            // Decode message
            Message msg;
            ProtocolMeta meta;

            Result decodeResult = protocol->decode(data, msg, meta);
            if (decodeResult != Result::Success) {
                m_circuitBreaker->recordFailure();
                return decodeResult;
            }

            // Update statistics
            updateStatistics(StatAction::MessageReceived);

            // Handle ACK if this is an ACK message
            if (meta.isAck) {
                handleIncomingAck(meta.token, true);
                m_circuitBreaker->recordSuccess();
                return Result::Success;
            }

            // Check if this message needs to be forwarded
            if (meta.isForwarded) {
                if (meta.finalDestination == getNodeId()) {
                    // Message has reached its final destination
                    return deliverToLocalSubscribers(msg);
                } else {
                    // Message needs to be forwarded
                    return forwardMessage(msg, meta);
                }
            } else {
                // Regular message for local delivery
                Result deliveryResult = deliverToLocalSubscribers(msg);

                // If delivery successful and message requires ACK, send it
                if (deliveryResult == Result::Success && meta.requiresAck) {
                    sendAck(meta.token, driverId, protocolId);
                }

                m_circuitBreaker->recordSuccess();
                return deliveryResult;
            }
        } catch (const std::exception& e) {
            // Log unexpected error
            m_circuitBreaker->recordFailure();
            return Result::UnexpectedError;
        }
    }

    // Set dependencies
    void setRouter(std::shared_ptr<IRouter> router) {
        m_router = std::move(router);
    }

    void setDriverManager(std::shared_ptr<DriverManager> driverManager) {
        m_driverManager = std::move(driverManager);
    }

    void setProtocolManager(std::shared_ptr<ProtocolManager> protocolManager) {
        m_protocolManager = std::move(protocolManager);
    }

    void setSubscriptionManager(std::shared_ptr<SubscriptionManager> subscriptionManager) {
        m_subscriptionManager = std::move(subscriptionManager);
    }

    void setQoSManager(std::shared_ptr<QoSManager> qosManager) {
        m_qosManager = std::move(qosManager);
    }

    void setOpcodeRegistry(std::shared_ptr<OpcodeRegistry> opcodeRegistry) {
        m_opcodeRegistry = std::move(opcodeRegistry);
    }

private:
    // Private constructor for singleton
    CommManager() : m_isRunning(false) {
        // Initialize statistics
        m_stats = {};
    }

    // Prevent copying or moving
    CommManager(const CommManager&) = delete;
    CommManager& operator=(const CommManager&) = delete;
    CommManager(CommManager&&) = delete;
    CommManager& operator=(CommManager&&) = delete;

    // Process message queues based on priority
    void processQueues() {
        // Deficit Round Robin scheduler weights
        const std::array<int, 4> weights = {16, 8, 4, 1}; // URGENT, HIGH, NORMAL, LOW
        std::array<int, 4> deficits = {0, 0, 0, 0};

        while (m_isRunning) {
            bool processedAny = false;

            // Process each priority queue with its weight
            for (int i = 0; i < 4; ++i) {
                deficits[i] += weights[i];

                // Process messages until deficit is used up
                while (deficits[i] > 0 && !m_priorityQueues[i].empty()) {
                    // Try to get a message
                    PriorityMessage msg;
                    if (!m_priorityQueues[i].pop(msg, 0)) {
                        break;
                    }

                    processedAny = true;
                    deficits[i]--;

                    // Process the message
                    processMessage(msg);
                }

                // If no messages processed, preserve deficit for next round
                if (m_priorityQueues[i].empty()) {
                    deficits[i] = 0;
                }
            }

            // If no messages were processed, sleep a bit to avoid spinning
            if (!processedAny) {
                OsClock::sleep(1);
            }
        }
    }

    // Process a single message from the queue
    void processMessage(const PriorityMessage& msg) {
        // 1. Resolve targets (subscribers/routes)
        std::vector<ForwardingTarget> targets = m_router->resolveOpcodeTargets(msg.opcode);

        if (targets.empty()) {
            // No subscribers, nothing to do
            m_qosManager->releaseResources(msg.context.priority);
            return;
        }

        // 2. Deliver/forward to each target
        bool anySuccess = false;

        for (const auto& target : targets) {
            Result result;

            if (target.isLocal) {
                // Local delivery
                Message localMsg{msg.opcode, std::span<const uint8_t>(msg.payload)};
                result = deliverToLocalSubscribers(localMsg);
            } else {
                // Forward to another node
                Message forwardMsg{msg.opcode, std::span<const uint8_t>(msg.payload)};
                result = forwardMessage(forwardMsg, target);
            }

            if (result == Result::Success) {
                anySuccess = true;
            }
        }

        // 3. Handle ACK if needed
        if (msg.context.ackPolicy == AckPolicy::RequireAck && msg.context.ackCallback) {
            // Store callback for later ACK
            if (anySuccess) {
                storeAckCallback(generateToken(), msg.context.ackCallback,
                               msg.context.timeoutMs ? msg.context.timeoutMs :
                               m_qosManager->getTimeoutForPriority(msg.context.priority));
            } else {
                // Immediate failure callback
                msg.context.ackCallback(false);
            }
        }

        // 4. Release QoS resources
        m_qosManager->releaseResources(msg.context.priority);
    }

    // Deliver message to local subscribers
    Result deliverToLocalSubscribers(const Message& msg) {
        try {
            m_subscriptionManager->deliverMessage(msg);
            return Result::Success;
        } catch (const std::exception& e) {
            // Log error
            return Result::DeliveryFailed;
        }
    }

    // Forward message to another node
    Result forwardMessage(const Message& msg, const ForwardingTarget& target) {
        // Get protocol
        IProtocol* protocol = m_protocolManager->getProtocol(target.protocolId);
        if (!protocol) {
            return Result::ProtocolNotFound;
        }

        // Get driver
        IDriver* driver = m_driverManager->getDriver(target.driverId);
        if (!driver) {
            return Result::DriverNotFound;
        }

        // Prepare forwarding metadata
        ForwardingMeta meta;
        meta.finalDestination = target.finalDestination;
        meta.remainingHops = target.remainingHops;
        meta.token = generateToken();
        meta.hopCount = 0;

        // Encode message
        std::vector<uint8_t> encodedData;
        Result encodeResult = protocol->encodeForward(msg, meta, encodedData);

        if (encodeResult != Result::Success) {
            return encodeResult;
        }

        // Send via driver
        Result sendResult = driver->send(encodedData);

        if (sendResult == Result::Success) {
            updateStatistics(StatAction::MessageForwarded);
        }

        return sendResult;
    }

    // Send ACK for a received message
    void sendAck(uint32_t token, std::string_view driverId, const std::string& protocolId) {
        // Get protocol
        IProtocol* protocol = m_protocolManager->getProtocol(protocolId);
        if (!protocol) {
            return;
        }

        // Get driver
        IDriver* driver = m_driverManager->getDriver(std::string(driverId));
        if (!driver) {
            return;
        }

        // Encode ACK
        std::vector<uint8_t> ackData;
        Result encodeResult = protocol->encodeAck(token, ackData);

        if (encodeResult != Result::Success) {
            return;
        }

        // Send ACK
        driver->send(ackData);

        updateStatistics(StatAction::AckSent);
    }

    // Handle incoming ACK
    void handleIncomingAck(uint32_t token, bool success) {
        // Find and execute the stored callback
        auto callback = getAckCallback(token);
        if (callback) {
            callback(success);
        }

        updateStatistics(StatAction::AckReceived);
    }

    // Store callback for later ACK
    void storeAckCallback(uint32_t token,
                         std::function<void(bool)> callback,
                         uint32_t timeoutMs) {
        // Store callback in map with expiry time
        std::lock_guard<std::mutex> lock(m_ackMutex);

        AckCallbackEntry entry;
        entry.callback = std::move(callback);
        entry.expiryTime = OsClock::nowMillis() + timeoutMs;

        m_pendingAcks[token] = std::move(entry);

        // Schedule cleanup after timeout
        std::thread([this, token, timeoutMs]() {
            OsClock::sleep(timeoutMs);
            handleAckTimeout(token);
        }).detach();
    }

    // Get and remove ACK callback
    std::function<void(bool)> getAckCallback(uint32_t token) {
        std::lock_guard<std::mutex> lock(m_ackMutex);

        auto it = m_pendingAcks.find(token);
        if (it != m_pendingAcks.end()) {
            auto callback = it->second.callback;
            m_pendingAcks.erase(it);
            return callback;
        }

        return nullptr;
    }

    // Handle ACK timeout
    void handleAckTimeout(uint32_t token) {
        std::lock_guard<std::mutex> lock(m_ackMutex);

        auto it = m_pendingAcks.find(token);
        if (it != m_pendingAcks.end() &&
            OsClock::nowMillis() >= it->second.expiryTime) {
            // Execute callback with failure
            if (it->second.callback) {
                it->second.callback(false);
            }

            // Remove entry
            m_pendingAcks.erase(it);

            updateStatistics(StatAction::AckTimeout);
        }
    }

    // Update statistics
    void updateStatistics(StatAction action, uint64_t value = 1) {
        std::lock_guard<std::mutex> lock(m_statsMutex);

        switch (action) {
            case StatAction::MessagePublished:
                m_stats.txMessages += value;
                break;
            case StatAction::MessageReceived:
                m_stats.rxMessages += value;
                break;
            case StatAction::MessageForwarded:
                m_stats.forwardedMessages += value;
                break;
            case StatAction::AckSent:
                m_stats.acksSent += value;
                break;
            case StatAction::AckReceived:
                m_stats.acksReceived += value;
                break;
            case StatAction::AckTimeout:
                m_stats.timeouts += value;
                break;
            case StatAction::Error:
                m_stats.errors += value;
                break;
        }
    }

    // Generate a unique token for message tracking
    uint32_t generateToken() {
        static std::atomic<uint32_t> tokenCounter{1};
        return tokenCounter.fetch_add(1, std::memory_order_relaxed);
    }

    // Get the protocol ID for a driver
    std::string getProtocolForDriver(const std::string& driverId) {
        // This would be determined from the route configuration
        // For now, return a default mapping
        if (driverId.find("ipc") != std::string::npos) {
            return "custom_lite";
        } else if (driverId.find("i2c") != std::string::npos) {
            return "custom_ack";
        } else if (driverId.find("spi") != std::string::npos) {
            return "custom_ack";
        } else {
            return "custom_ack";
        }
    }

    // Get the current node ID
    std::string getNodeId() const {
        // In a real implementation, this would be configured
        return "CurrentNode";
    }

    // Internal structures
    struct PriorityMessage {
        uint16_t opcode;
        std::vector<uint8_t> payload;
        MessageContext context;
        uint64_t timestamp;
    };

    struct AckCallbackEntry {
        std::function<void(bool)> callback;
        uint64_t expiryTime;
    };

    enum class StatAction {
        MessagePublished,
        MessageReceived,
        MessageForwarded,
        AckSent,
        AckReceived,
        AckTimeout,
        Error
    };

    // Member variables
    std::shared_ptr<OpcodeRegistry> m_opcodeRegistry;
    std::shared_ptr<IRouter> m_router;
    std::shared_ptr<DriverManager> m_driverManager;
    std::shared_ptr<ProtocolManager> m_protocolManager;
    std::shared_ptr<SubscriptionManager> m_subscriptionManager;
    std::shared_ptr<QoSManager> m_qosManager;
    std::array<OsQueue<PriorityMessage>, 4> m_priorityQueues;
    std::atomic<bool> m_isRunning;
    std::thread m_processingThread;
    mutable std::mutex m_statsMutex;
    Stats m_stats;
    std::shared_ptr<CircuitBreaker> m_circuitBreaker;
    std::unordered_map<uint32_t, AckCallbackEntry> m_pendingAcks;
    std::mutex m_ackMutex;

    // Constants
    static constexpr uint64_t INVALID_SUBSCRIPTION_ID = 0;
};
```

## OpcodeRegistry Implementation

```cpp name=opcode_registry.hpp
class OpcodeRegistry {
public:
    OpcodeRegistry() = default;

    // Register a new OPCODE with metadata
    Result registerOpcode(const OpcodeInfo& info) {
        std::lock_guard<std::mutex> lock(m_mutex);

        // Check if already registered
        if (m_opcodes.count(info.opcode) > 0) {
            if (!overwriteAllowed(info.opcode)) {
                return Result::OpcodeAlreadyRegistered;
            }
        }

        // Validate OPCODE range
        if (!validateOpcodeRange(info.opcode, info.name)) {
            return Result::InvalidOpcodeRange;
        }

        // Validate payload size
        if (info.maxPayloadSize > MAX_ALLOWED_PAYLOAD) {
            return Result::PayloadSizeTooLarge;
        }

        // Check for reserved names
        if (isReservedName(info.name)) {
            return Result::ReservedOpcodeName;
        }

        // Store OPCODE
        m_opcodes[info.opcode] = info;
        m_opcodesByName[info.name] = info.opcode;

        // Notify subscribers of registry changes
        if (m_registryUpdateCallback) {
            m_registryUpdateCallback(info);
        }

        return Result::Success;
    }

    // Get OPCODE info by OPCODE number
    std::optional<OpcodeInfo> getOpcodeInfo(uint16_t opcode) const {
        std::lock_guard<std::mutex> lock(m_mutex);

        // Check cache first
        auto cacheIt = m_commonOpcodeCache.find(opcode);
        if (cacheIt != m_commonOpcodeCache.end()) {
            return cacheIt->second;
        }

        // Regular lookup
        auto it = m_opcodes.find(opcode);
        if (it == m_opcodes.end()) {
            return std::nullopt;
        }

        return it->second;
    }

    // Get OPCODE number by name
    std::optional<uint16_t> getOpcodeByName(const std::string& name) const {
        std::lock_guard<std::mutex> lock(m_mutex);

        auto it = m_opcodesByName.find(name);
        if (it == m_opcodesByName.end()) {
            return std::nullopt;
        }

        return it->second;
    }

    // Get all registered OPCODEs
    std::vector<OpcodeInfo> getAllOpcodes() const {
        std::lock_guard<std::mutex> lock(m_mutex);

        std::vector<OpcodeInfo> result;
        result.reserve(m_opcodes.size());

        for (const auto& [_, info] : m_opcodes) {
            result.push_back(info);
        }

        return result;
    }

    // Faster validation with output parameter
    bool validateOpcodeAndGetInfo(uint16_t opcode, OpcodeInfo& outInfo) {
        std::lock_guard<std::mutex> lock(m_mutex);

        // Fast path: check common OPCODEs first
        auto cacheIt = m_commonOpcodeCache.find(opcode);
        if (cacheIt != m_commonOpcodeCache.end()) {
            outInfo = cacheIt->second;
            return true;
        }

        // Normal lookup
        auto it = m_opcodes.find(opcode);
        if (it == m_opcodes.end()) {
            return false;
        }

        outInfo = it->second;

        // Update usage statistics
        updateUsageStatistics(opcode);

        // Update cache if this is becoming a common OPCODE
        if (isCommonOpcode(opcode)) {
            m_commonOpcodeCache[opcode] = outInfo;
        }

        return true;
    }

    // Mark an OPCODE as deprecated
    void deprecateOpcode(uint16_t opcode, uint16_t replacementOpcode = 0) {
        std::lock_guard<std::mutex> lock(m_mutex);

        auto it = m_opcodes.find(opcode);
        if (it != m_opcodes.end()) {
            it->second.deprecated = true;

            if (replacementOpcode != 0) {
                it->second.description += " [DEPRECATED: Use 0x" +
                    toHexString(replacementOpcode) + " instead]";
            }

            // Remove from cache if present
            m_commonOpcodeCache.erase(opcode);

            // Notify about update
            if (m_registryUpdateCallback) {
                m_registryUpdateCallback(it->second);
            }
        }
    }

    // Register callback for OPCODE registry changes
    void registerOpcodeUpdateCallback(std::function<void(const OpcodeInfo&)> callback) {
        std::lock_guard<std::mutex> lock(m_mutex);
        m_registryUpdateCallback = std::move(callback);
    }

    // Batch register multiple OPCODEs
    Result batchRegister(const std::vector<OpcodeInfo>& opcodes) {
        std::lock_guard<std::mutex> lock(m_mutex);

        // First validate all OPCODEs
        for (const auto& info : opcodes) {
            // Check if already registered
            if (m_opcodes.count(info.opcode) > 0 && !overwriteAllowed(info.opcode)) {
                return Result::OpcodeAlreadyRegistered;
            }

            // Validate OPCODE range
            if (!validateOpcodeRange(info.opcode, info.name)) {
                return Result::InvalidOpcodeRange;
            }

            // Validate payload size
            if (info.maxPayloadSize > MAX_ALLOWED_PAYLOAD) {
                return Result::PayloadSizeTooLarge;
            }

            // Check for reserved names
            if (isReservedName(info.name)) {
                return Result::ReservedOpcodeName;
            }
        }

        // Now register all OPCODEs
        for (const auto& info : opcodes) {
            m_opcodes[info.opcode] = info;
            m_opcodesByName[info.name] = info.opcode;

            // Notify subscribers of registry changes
            if (m_registryUpdateCallback) {
                m_registryUpdateCallback(info);
            }
        }

        return Result::Success;
    }

private:
    static constexpr uint32_t MAX_ALLOWED_PAYLOAD = 65535;
    static constexpr uint32_t COMMON_OPCODE_THRESHOLD = 100;

    std::unordered_map<uint16_t, OpcodeInfo> m_opcodes;
    std::unordered_map<std::string, uint16_t> m_opcodesByName;
    mutable std::unordered_map<uint16_t, OpcodeInfo> m_commonOpcodeCache;
    mutable std::mutex m_mutex;
    std::unordered_map<uint16_t, uint64_t> m_usageCount;
    std::function<void(const OpcodeInfo&)> m_registryUpdateCallback;

    // Check if an OPCODE is in a valid range
    bool validateOpcodeRange(uint16_t opcode, const std::string& name) {
        // Example validation rules:
        // - System OPCODEs: 0x0001-0x00FF
        // - Audio OPCODEs: 0x0100-0x01FF
        // - BLE OPCODEs: 0x0200-0x02FF
        // - Sensor OPCODEs: 0x0300-0x03FF
        // - Application OPCODEs: 0x1000-0x1FFF

        // In a real implementation, this would check against configured ranges
        return opcode != 0; // Just ensure non-zero for this example
    }

    // Track OPCODE usage for cache optimization
    void updateUsageStatistics(uint16_t opcode) {
        m_usageCount[opcode]++;
    }

    // Determine if an OPCODE is commonly used
    bool isCommonOpcode(uint16_t opcode) {
        return m_usageCount[opcode] > COMMON_OPCODE_THRESHOLD;
    }

    // Check if an OPCODE name is reserved
    bool isReservedName(const std::string& name) {
        // Example reserved names
        static const std::unordered_set<std::string> reservedNames = {
            "RESERVED", "SYSTEM", "TEST"
        };

        return reservedNames.find(name) != reservedNames.end();
    }

    // Check if an OPCODE can be overwritten
    bool overwriteAllowed(uint16_t opcode) {
        // Example: Allow overwriting test OPCODEs (0xF000-0xFFFF)
        return (opcode >= 0xF000);
    }

    // Convert number to hex string
    static std::string toHexString(uint16_t value) {
        std::stringstream ss;
        ss << std::hex << std::setfill('0') << std::setw(4) << value;
        return ss.str();
    }
};
```

## Enumeration Types

```cpp name=messaging_types.hpp
#ifndef MSG_TYPES_HPP
#define MSG_TYPES_HPP

namespace msg {

/**
 * @brief Priority levels for QoS. URGENT preempts others, with starvation guard via deficit round-robin.
 * @note LOW for background, URGENT for critical (e.g., alerts). Independent of hardware—queues abstract delays.
 */
enum class Priority {
    DEFAULT = 0, // Use OPCODE default
    LOW = 0,     // Background tasks, non-time-sensitive
    NORMAL = 1,  // Regular operation, standard priority
    HIGH = 2,    // Important messages, prioritized handling
    URGENT = 3   // Critical messages, highest priority
};

/**
 * @brief ACK policies. RequireAck ensures end-to-end delivery via tokens, even multi-hop.
 * @note None for fire-and-forget (e.g., telemetry). Thread-safe, with jittered retries.
 */
enum class AckPolicy {
    DEFAULT = 0, // Use OPCODE default
    None = 0,    // No acknowledgment required
    RequireAck = 1 // Require acknowledgment for delivery confirmation
};

/**
 * @brief Results. QueueFull triggers backpressure—caller retries or drops.
 */
enum class Result {
    Success = 0,
    QueueFull,
    InvalidOpcode,
    Timeout,
    ConfigurationInvalid,
    PayloadTooLarge,
    InvalidFormat,
    CrcError,
    NotInitialized,
    HardwareError,
    DriverNotFound,
    ProtocolNotFound,
    CircuitBreakerOpen,
    SubscriptionNotFound,
    ResourceAllocationFailed,
    DeliveryFailed,
    InvalidOpcodeRange,
    OpcodeAlreadyRegistered,
    PayloadSizeTooLarge,
    ReservedOpcodeName,
    NotSupported,
    InvalidSize,
    UnexpectedError,
    MissingDependency,
    AlreadyInitialized
};

/**
 * @brief Stats for diagnostics. Errors include CRC fails, timeouts.
 * @note Query periodically for health monitoring. Platform-independent.
 */
struct Stats {
    uint64_t txMessages = 0;
    uint64_t rxMessages = 0;
    uint64_t forwardedMessages = 0;
    uint64_t acksSent = 0;
    uint64_t acksReceived = 0;
    uint64_t errors = 0;
    uint64_t timeouts = 0;
    uint32_t avgLatencyUs = 0;
};

/**
 * @struct OpcodeInfo
 * @brief OPCODE metadata.
 */
struct OpcodeInfo {
    uint16_t opcode;
    std::string name;
    std::string description;
    uint32_t maxPayloadSize;
    Priority defaultPriority;
    AckPolicy defaultAckPolicy;
    uint32_t version;
    bool deprecated = false;
};

/**
 * @struct ForwardingMeta
 * @brief Metadata for message forwarding
 */
struct ForwardingMeta {
    std::string finalDestination;
    std::vector<std::string> remainingHops;
    uint32_t token;
    uint8_t hopCount;
    bool requiresAck;
    Priority priority;
};

// Type for subscription IDs
using SubscriptionId = uint64_t;

} // namespace msg

#endif // MSG_TYPES_HPP
```

## Configuration Structure Definitions

```cpp name=config_types.hpp
#ifndef MSG_CONFIG_TYPES_HPP
#define MSG_CONFIG_TYPES_HPP

#include <string>
#include <vector>
#include <map>
#include <unordered_map>
#include "messaging_types.hpp"

namespace msg {

/**
 * @struct DriverConfig
 * @brief Driver setup.
 */
struct DriverConfig {
    std::string id;
    std::string type;    // e.g., "IPC"
    std::string variant; // e.g., "OpenAMP"
    std::unordered_map<std::string, std::string> settings;
    std::unordered_map<std::string, std::string> capabilities;
};

/**
 * @struct ProtocolConfig
 * @brief Protocol setup.
 */
struct ProtocolConfig {
    std::string id;
    std::string type; // e.g., "CUSTOM_ACK"
    std::unordered_map<std::string, std::string> settings;
};

/**
 * @struct RouteConfig
 * @brief Route setup.
 */
struct RouteConfig {
    std::string id;
    std::string src;
    std::string dst;
    std::string driverId;
    std::string protocolId;
    Priority priority = Priority::NORMAL;
    AckPolicy ackPolicy = AckPolicy::None;
    std::vector<std::string> requiresCaps;
    bool bidirectional = false;
};

/**
 * @struct TopologyPath
 * @brief Multi-hop path.
 */
struct TopologyPath {
    std::string destination;
    std::vector<std::string> hops;
    uint32_t totalCost;
    std::vector<std::string> backup;
};

/**
 * @struct QoSProfile
 * @brief QoS settings for a priority level
 */
struct QoSProfile {
    std::string id;
    std::string description;
    uint32_t urgentTimeoutMs;
    uint32_t highRetryMs;
    uint32_t normalRetryMs;
    uint32_t lowRetryMs;

    struct ResourceReservation {
        uint8_t cpuPercentage;
        uint32_t memoryKb;
        bool priorityBoost;
    } resourceReservation;

    struct CircuitBreaker {
        uint32_t errorThreshold;
        uint32_t windowMs;
        uint32_t tripTimeMs;
    } circuitBreaker;
};

/**
 * @struct SystemConfig
 * @brief Full config.
 */
struct SystemConfig {
    std::string nodeId;
    std::vector<DriverConfig> drivers;
    std::vector<ProtocolConfig> protocols;
    std::vector<RouteConfig> routes;
    std::vector<TopologyPath> topologyPaths;
    std::vector<QoSProfile> qosProfiles;
    std::string hash; // Validation hash
};

/**
 * @struct ValidationResult
 * @brief Result of config validation
 */
struct ValidationResult {
    bool isValid;
    std::string errorMessage;
    std::string errorLocation;
};

/**
 * @struct DriverCapabilities
 * @brief Driver features.
 */
struct DriverCapabilities {
    uint32_t maxFrameSize;
    bool supportsDMA;
    bool supportsAsync;
    uint32_t latencyUs;
    std::unordered_map<std::string, std::string> additionalCaps;
};

/**
 * @struct ProtocolCapabilities
 * @brief Protocol features
 */
struct ProtocolCapabilities {
    bool supportsAck;
    bool supportsFragmentation;
    bool supportsSecurity;
    uint32_t maxMessageSize;
    uint32_t overhead;
};

} // namespace msg

#endif // MSG_CONFIG_TYPES_HPP
```

## Usage Example

```cpp name=usage_example.cpp
#include "comm_manager.hpp"
#include "opcode_registry.hpp"
#include "config_manager.hpp"
#include <iostream>
#include <vector>

using namespace msg;

void setupMessagingSystem() {
    // Get CommManager instance
    auto& commMgr = CommManager::getInstance();

    // Create and register dependencies
    auto opcodeRegistry = std::make_shared<OpcodeRegistry>();
    auto router = std::make_shared<RouteManager>("SoC1_Core2");
    auto driverManager = std::make_shared<DriverManager>();
    auto protocolManager = std::make_shared<ProtocolManager>();
    auto subscriptionManager = std::make_shared<SubscriptionManager>();
    auto qosManager = std::make_shared<QoSManager>();

    // Set up factories
    auto driverFactory = std::make_shared<StandardDriverFactory>();
    driverManager->setDriverFactory(driverFactory);

    auto protocolFactory = std::make_shared<StandardProtocolFactory>();
    protocolManager->setProtocolFactory(protocolFactory);

    // Connect components
    commMgr.setOpcodeRegistry(opcodeRegistry);
    commMgr.setRouter(router);
    commMgr.setDriverManager(driverManager);
    commMgr.setProtocolManager(protocolManager);
    commMgr.setSubscriptionManager(subscriptionManager);
    commMgr.setQoSManager(qosManager);

    subscriptionManager->setOpcodeRegistry(opcodeRegistry);
    subscriptionManager->setRouteManager(router);

    router->setDriverManager(driverManager);

    // Initialize the configuration
    ConfigManager configMgr(commMgr);
    Result result = configMgr.loadConfig("soc1_core2.json");

    if (result != Result::Success) {
        std::cerr << "Failed to load configuration: " << static_cast<int>(result) << std::endl;
        return;
    }

    // Register standard OPCODEs
    opcodeRegistry->registerOpcode({
        0x0001, "SYS_HEARTBEAT", "System heartbeat message",
        16, Priority::LOW, AckPolicy::None, 1, false
    });

    opcodeRegistry->registerOpcode({
        0x0002, "SYS_POWER_STATE", "Power state change notification",
        8, Priority::HIGH, AckPolicy::RequireAck, 1, false
    });

    opcodeRegistry->registerOpcode({
        0x0100, "AUDIO_PCM_STREAM", "Raw PCM audio data",
        4096, Priority::URGENT, AckPolicy::None, 1, false
    });

    opcodeRegistry->registerOpcode({
        0x0300, "SENSOR_ACCEL_DATA", "Accelerometer sensor data",
        64, Priority::NORMAL, AckPolicy::None, 1, false
    });

    // Initialize the system
    result = commMgr.initialize();

    if (result != Result::Success) {
        std::cerr << "Failed to initialize CommManager: " << static_cast<int>(result) << std::endl;
        return;
    }

    std::cout << "Messaging system initialized successfully" << std::endl;
}

void publishSensorData() {
    auto& commMgr = CommManager::getInstance();

    // Create sensor data
    std::vector<uint8_t> sensorData = {0x01, 0x02, 0x03, 0x04, 0x05, 0x06};

    // Create message
    Message msg{0x0300, sensorData};

    // Set up context with ACK
    MessageContext ctx;
    ctx.priority = Priority::NORMAL;
    ctx.ackPolicy = AckPolicy::RequireAck;
    ctx.ackCallback = [](bool success) {
        if (success) {
            std::cout << "Sensor data delivered successfully" << std::endl;
        } else {
            std::cout << "Failed to deliver sensor data" << std::endl;
        }
    };

    // Publish message
    Result result = commMgr.publish(msg, ctx);

    if (result != Result::Success) {
        std::cerr << "Failed to publish sensor data: " << static_cast<int>(result) << std::endl;
    }
}

void setupSubscriptions() {
    auto& commMgr = CommManager::getInstance();

    // Subscribe to heartbeat messages
    commMgr.subscribe(0x0001, [](const Message& msg) {
        std::cout << "Received heartbeat message" << std::endl;
    });

    // Subscribe to power state changes
    commMgr.subscribe(0x0002, [](const Message& msg) {
        if (msg.payload.size() >= 1) {
            uint8_t powerState = msg.payload[0];
            std::cout << "Power state changed to: " << static_cast<int>(powerState) << std::endl;

            // React to power state change
            switch (powerState) {
                case 0: std::cout << "  System powering down" << std::endl; break;
                case 1: std::cout << "  System powering up" << std::endl; break;
                case 2: std::cout << "  System entering low power mode" << std::endl; break;
                default: std::cout << "  Unknown power state" << std::endl; break;
            }
        }
    });

    // Subscribe to sensor data
    commMgr.subscribe(0x0300, [](const Message& msg) {
        std::cout << "Received sensor data: ";
        for (uint8_t byte : msg.payload) {
            std::cout << std::hex << std::setw(2) << std::setfill('0')
                      << static_cast<int>(byte) << " ";
        }
        std::cout << std::dec << std::endl;
    });
}

int main() {
    // Initialize the messaging system
    setupMessagingSystem();

    // Set up subscriptions
    setupSubscriptions();

    // Publish some data
    publishSensorData();

    // In a real application, you would keep the program running
    std::cout << "Press Enter to exit..." << std::endl;
    std::cin.get();

    // Clean up
    CommManager::getInstance().stop();

    return 0;
}
```

## Configuration Example (JSON)

```json name=soc1_core2.json
{
  "nodeId": "SoC1_Core2",
  "drivers": [
    {
      "id": "ipc_soc1",
      "type": "IPC",
      "variant": "Mailbox",
      "settings": {
        "queueSize": "64",
        "sharedMemSize": "32768",
        "interruptLine": "7"
      },
      "capabilities": {
        "maxFrame": "1024",
        "async": "true",
        "dma": "false",
        "latency": "50"
      }
    },
    {
      "id": "i2c_to_soc2",
      "type": "I2C",
      "variant": "STM32HAL",
      "settings": {
        "interface": "I2C1",
        "speed": "400000",
        "address": "0x42",
        "pullups": "enabled"
      },
      "capabilities": {
        "maxFrame": "256",
        "dma": "true",
        "latency": "200",
        "concurrentTransactions": "false"
      }
    },
    {
      "id": "spi_to_soc3",
      "type": "SPI",
      "variant": "STM32HAL",
      "settings": {
        "interface": "SPI1",
        "speed": "10000000",
        "mode": "0",
        "dataWidth": "1"
      },
      "capabilities": {
        "maxFrame": "512",
        "async": "true",
        "dma": "true",
        "latency": "100"
      }
    }
  ],
  "protocols": [
    {
      "id": "custom_ack",
      "type": "CUSTOM_ACK",
      "settings": {
        "ackTimeout": "500",
        "maxRetries": "3",
        "backoffMultiplier": "2.0",
        "enableCRC": "true"
      }
    },
    {
      "id": "custom_lite",
      "type": "CUSTOM_LITE",
      "settings": {
        "enableCRC": "true",
        "headerCompression": "true"
      }
    }
  ],
  "routes": [
    {
      "id": "route_internal_ipc",
      "src": "SoC1_Core1",
      "dst": "SoC1_Core2",
      "driverId": "ipc_soc1",
      "protocolId": "custom_lite",
      "priority": "NORMAL",
      "ackPolicy": "None",
      "requiresCaps": ["maxFrame>=512"],
      "bidirectional": true
    },
    {
      "id": "route_to_soc2",
      "src": "SoC1_Core2",
      "dst": "SoC2_Core1",
      "driverId": "i2c_to_soc2",
      "protocolId": "custom_ack",
      "priority": "HIGH",
      "ackPolicy": "RequireAck",
      "requiresCaps": ["dma==true", "latency<300"],
      "bidirectional": true
    },
    {
      "id": "route_to_soc3",
      "src": "SoC1_Core2",
      "dst": "SoC3_Core1",
      "driverId": "spi_to_soc3",
      "protocolId": "custom_ack",
      "priority": "URGENT",
      "ackPolicy": "RequireAck",
      "requiresCaps": ["async==true", "maxFrame>256"],
      "bidirectional": true
    }
  ],
  "topologyPaths": [
    {
      "destination": "SoC2_Core2",
      "hops": ["route_to_soc2"],
      "totalCost": 10,
      "backup": []
    },
    {
      "destination": "SoC3_Core1",
      "hops": ["route_to_soc3"],
      "totalCost": 15,
      "backup": []
    }
  ],
  "qosProfiles": [
    {
      "id": "AUDIO_STREAM",
      "description": "Real-time audio streaming profile",
      "urgentTimeoutMs": 10,
      "highRetryMs": 5,
      "normalRetryMs": 20,
      "lowRetryMs": 50,
      "resourceReservation": {
        "cpuPercentage": 30,
        "memoryKb": 1024,
        "priorityBoost": true
      },
      "circuitBreaker": {
        "errorThreshold": 10,
        "windowMs": 1000,
        "tripTimeMs": 5000
      }
    },
    {
      "id": "SYSTEM_CRITICAL",
      "description": "Critical system operations",
      "urgentTimeoutMs": 500,
      "highRetryMs": 100,
      "normalRetryMs": 200,
      "lowRetryMs": 500,
      "resourceReservation": {
        "cpuPercentage": 10,
        "memoryKb": 512,
        "priorityBoost": true
      },
      "circuitBreaker": {
        "errorThreshold": 5,
        "windowMs": 2000,
        "tripTimeMs": 10000
      }
    }
  ],
  "hash": "sha256:1a2b3c4d5e6f7890abcdef1234567890"
}
```

## Summary of Design Benefits

This OPCODE-based messaging system architecture provides:

1. **Decoupled Components**: Publishers and subscribers are completely decoupled, communicating only via OPCODEs

2. **Platform Independence**: The architecture abstracts away platform-specific details, allowing operation across different SoCs and OS environments

3. **Scalability**: Designed to handle complex multi-SoC systems with heterogeneous cores and communication paths

4. **Reliability**: Multiple QoS levels, acknowledgment policies, and circuit breakers ensure robust message delivery

5. **Configurability**: JSON-based configuration allows for hot-swapping components without code changes

6. **Performance**: Optimized critical paths with caching, prioritization, and efficient memory handling

7. **Extensibility**: Interfaces allow for new driver, protocol, and routing implementations to be added without changing core logic

8. **Security**: Support for secure protocols and access control based on OPCODE ranges

The implementation follows C++ best practices with SOLID principles, proper memory management, and thread safety considerations appropriate for embedded systems.
