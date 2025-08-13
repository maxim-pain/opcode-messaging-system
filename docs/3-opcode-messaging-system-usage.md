# OPCODE Messaging System - Usage Examples & Best Practices

## 1. Usage Examples

### 1.1 System Initialization

This example demonstrates how to initialize the messaging system, load configuration, and register OPCODEs:

```cpp
#include "msg/comm.hpp"
#include "msg/config.hpp"
#include "msg/opcode_registry.hpp"

void initializeSystem() {
    // 1. Load configuration
    ConfigManager configMgr;
    auto result = configMgr.loadConfig("soc1_core2.json");
    if (result != Result::Success) {
        // Handle error
        return;
    }

    // 2. Register OPCODEs
    OpcodeRegistry registry;
    registry.registerOpcode({
        0x1001,                    // OPCODE
        "SENSOR_DATA",             // Name
        "Sensor telemetry data",   // Description
        512,                       // Max payload size
        Priority::HIGH,            // Default priority
        AckPolicy::RequireAck,     // Default ACK policy
        1,                         // Version
        false                      // Not deprecated
    });

    registry.registerOpcode({
        0x1002,
        "CONTROL_CMD",
        "Control commands",
        64,
        Priority::URGENT,
        AckPolicy::RequireAck,
        1,
        false
    });

    // 3. Create CommManager (singleton)
    CommManager& commMgr = CommManager::getInstance();

    // System is now ready for messaging
}
```

### 1.2 Publishing Messages

This example shows how to publish sensor data with acknowledgment:

```cpp
void publishSensorData() {
    CommManager& cm = CommManager::getInstance();

    // Prepare sensor data
    std::vector<uint8_t> sensorData(256);
    // ... fill data ...

    // Create message with OPCODE
    Message msg{0x1001, sensorData};

    // Create context with options
    MessageContext ctx;
    ctx.priority = Priority::HIGH;
    ctx.ackPolicy = AckPolicy::RequireAck;
    ctx.ackCallback = [](bool success) {
        if (success) {
            // Message successfully delivered to all subscribers
            logSuccess("Sensor data delivered");
        } else {
            // Delivery failed after retries
            logError("Failed to deliver sensor data");
        }
    };

    // Publish (routes automatically to subscribers across SoCs/cores)
    auto result = cm.publish(msg, ctx);
    if (result != Result::Success) {
        // Handle error (e.g., queue full, retry later)
        switch (result) {
            case Result::QueueFull:
                applyBackpressure();
                scheduleRetry(msg, ctx, 100); // retry after 100ms
                break;
            case Result::InvalidOpcode:
                logError("Invalid OPCODE: 0x1001");
                break;
            default:
                logError("Unknown error when publishing");
                break;
        }
    }
}
```

### 1.3 Subscribing to Messages

This example demonstrates how to subscribe to multiple message types:

```cpp
void setupSubscriptions() {
    CommManager& cm = CommManager::getInstance();

    // Subscribe to SENSOR_DATA
    cm.subscribe(0x1001, [](const Message& msg) {
        // Process incoming sensor data
        std::cout << "Received sensor data, size: " << msg.payload.size() << " bytes" << std::endl;

        // Parse data (assuming specific format)
        SensorReading reading;
        if (parseSensorData(msg.payload.data(), msg.payload.size(), reading)) {
            processSensorReading(reading);
        }
    });

    // Subscribe to CONTROL_CMD
    cm.subscribe(0x1002, [](const Message& msg) {
        // Process control command
        std::cout << "Received control command" << std::endl;

        // Parse command
        ControlCommand cmd;
        if (parseControlCommand(msg.payload.data(), msg.payload.size(), cmd)) {
            executeCommand(cmd);
        }
    });

    // Subscribe to system heartbeat for health monitoring
    cm.subscribe(0x0001, [](const Message& msg) {
        // Update system health status
        updateLastHeartbeatTime();
    });
}
```

### 1.4 Configuration Hot-Swap

This example shows how to perform hot-swap of the configuration at runtime:

```cpp
void performHotSwap() {
    ConfigManager configMgr;

    // Create new configuration (e.g., switch I2C driver variant for better performance)
    SystemConfig newConfig;
    newConfig.nodeId = "SoC1_Core2";

    // ... setup drivers, protocols, routes ...

    // Example: Update I2C driver to NXP variant
    DriverConfig i2cConfig;
    i2cConfig.id = "i2c_to_soc2";
    i2cConfig.type = "I2C";
    i2cConfig.variant = "NXP_LPI2C";  // Changed from STM32HAL
    i2cConfig.settings["interface"] = "I2C1";
    i2cConfig.settings["speed"] = "1000000";  // Faster speed
    i2cConfig.capabilities["maxFrame"] = "256";
    i2cConfig.capabilities["dma"] = "true";
    i2cConfig.capabilities["latency"] = "150";  // Better latency

    newConfig.drivers.push_back(i2cConfig);

    // Add protocol configurations
    ProtocolConfig ackProtocol;
    ackProtocol.id = "custom_ack";
    ackProtocol.type = "CUSTOM_ACK";
    ackProtocol.settings["ackTimeout"] = "300";  // Reduced from 500ms
    ackProtocol.settings["maxRetries"] = "5";    // Increased from 3
    newConfig.protocols.push_back(ackProtocol);

    // Add route configurations
    RouteConfig route;
    route.id = "route_to_soc2";
    route.src = "SoC1_Core2";
    route.dst = "SoC2_Core1";
    route.driverId = "i2c_to_soc2";
    route.protocolId = "custom_ack";
    route.priority = Priority::HIGH;
    route.ackPolicy = AckPolicy::RequireAck;
    route.requiresCaps = {"dma==true", "latency<300"};
    newConfig.routes.push_back(route);

    // Compute hash for integrity
    newConfig.hash = "sha256:updated_hash_value";

    // Apply hot-swap (atomic, with rollback on failure)
    auto result = configMgr.hotSwapConfig(newConfig);
    if (result == Result::Success) {
        std::cout << "Configuration hot-swapped successfully" << std::endl;
        notifySystemComponents();
    } else {
        std::cout << "Hot-swap failed, rolled back to previous configuration" << std::endl;
        logHotSwapFailure(result);
    }
}
```

### 1.5 Handling Large Messages with Fragmentation

This example demonstrates how to publish large audio data that will be automatically fragmented:

```cpp
void sendAudioStream() {
    CommManager& cm = CommManager::getInstance();

    // Large audio buffer (>MTU size, will be fragmented)
    std::vector<uint8_t> audioBuffer(8192);  // 8KB of audio data
    fillAudioBuffer(audioBuffer);  // Fill with PCM data

    // Create message with AUDIO_PCM_STREAM opcode
    Message msg{0x0100, audioBuffer};

    // Configure for streaming audio (high priority, no ACK for performance)
    MessageContext ctx;
    ctx.priority = Priority::URGENT;
    ctx.ackPolicy = AckPolicy::None;

    // System will automatically:
    // 1. Fragment the data based on driver MTU size
    // 2. Use the appropriate protocol (custom_lite for audio)
    // 3. Route via the configured path (e.g., DSP_Core to ARM_Core via IPC)
    // 4. Reassemble at destination before delivering to subscribers
    auto result = cm.publish(msg, ctx);

    if (result != Result::Success) {
        handleAudioStreamError(result);
    }
}
```

### 1.6 Multi-Core Integration Example

This example shows integration across multiple cores for a voice command feature:

```cpp
// On DSP Core: Audio processing module
void setupDspAudioModule() {
    CommManager& cm = CommManager::getInstance();

    // Listen for audio configuration changes
    cm.subscribe(0x0101, [](const Message& msg) {
        AudioConfig config;
        parseAudioConfig(msg.payload.data(), msg.payload.size(), config);
        applyAudioConfig(config);
    });

    // Process audio and detect keywords
    startAudioProcessing([](const KeywordDetection& keyword) {
        // When keyword detected, notify main processor
        std::vector<uint8_t> data(sizeof(KeywordDetection));
        memcpy(data.data(), &keyword, sizeof(KeywordDetection));

        Message msg{0x0102, data};  // KEYWORD_DETECTED opcode
        MessageContext ctx{Priority::HIGH, AckPolicy::RequireAck};

        CommManager::getInstance().publish(msg, ctx);
    });
}

// On ARM Core: Command processor
void setupArmCommandProcessor() {
    CommManager& cm = CommManager::getInstance();

    // Listen for keyword detections from DSP
    cm.subscribe(0x0102, [](const Message& msg) {
        KeywordDetection keyword;
        memcpy(&keyword, msg.payload.data(), std::min(msg.payload.size(), sizeof(KeywordDetection)));

        // Start full voice recognition
        startVoiceRecognition();

        // Notify BLE Core about voice command mode
        std::vector<uint8_t> data = {1};  // 1 = active
        Message stateMsg{0x0515, data};   // VOICE_CMD_STATE opcode
        cm.publish(stateMsg);
    });

    // Send audio configuration to DSP
    AudioConfig config = createDefaultAudioConfig();
    std::vector<uint8_t> data(sizeof(AudioConfig));
    memcpy(data.data(), &config, sizeof(AudioConfig));

    Message configMsg{0x0101, data};
    cm.publish(configMsg, {Priority::NORMAL, AckPolicy::RequireAck});
}
```

## 2. Best Practices for Managers

### 2.1 ConfigManager Best Practices

The ConfigManager handles loading, validation, and hot-swapping of system configurations:

```cpp
class BestConfigManager {
public:
    // Validate configuration before applying
    bool validateConfig(const SystemConfig& config) {
        // 1. Validate basic integrity
        if (config.nodeId.empty() || config.drivers.empty() || config.hash.empty()) {
            logError("Missing required fields");
            return false;
        }

        // 2. Verify hash
        std::string computedHash = computeConfigHash(config);
        if (computedHash != config.hash) {
            logError("Config hash mismatch: possible corruption");
            return false;
        }

        // 3. Check for circular dependencies in routes/paths
        if (hasCircularDependencies(config.topologyPaths)) {
            logError("Circular dependencies detected in topology paths");
            return false;
        }

        // 4. Validate driver capabilities match route requirements
        for (const auto& route : config.routes) {
            auto driverIt = findDriverById(config.drivers, route.driverId);
            if (driverIt == config.drivers.end()) {
                logError("Route references non-existent driver: " + route.driverId);
                return false;
            }

            // Check each capability requirement
            for (const auto& reqCap : route.requiresCaps) {
                if (!meetsCapabilityRequirement(driverIt->capabilities, reqCap)) {
                    logError("Driver " + route.driverId + " does not meet requirement: " + reqCap);
                    return false;
                }
            }
        }

        return true;
    }

    // Safely apply configuration with atomic update and rollback
    Result applyConfigSafely(const SystemConfig& config) {
        // 1. Backup current configuration
        SystemConfig backup = m_currentConfig;

        try {
            // 2. Create driver instances first but don't activate
            std::vector<std::unique_ptr<IDriver>> newDrivers;
            for (const auto& driverCfg : config.drivers) {
                auto driver = createDriver(driverCfg);
                if (!driver) {
                    throw std::runtime_error("Failed to create driver: " + driverCfg.id);
                }
                newDrivers.push_back(std::move(driver));
            }

            // 3. Initialize drivers (but keep old ones running)
            for (auto& driver : newDrivers) {
                auto result = driver->initialize();
                if (result != Result::Success) {
                    throw std::runtime_error("Failed to initialize driver");
                }
            }

            // 4. Create protocol instances
            std::vector<std::unique_ptr<IProtocol>> newProtocols;
            for (const auto& protocolCfg : config.protocols) {
                auto protocol = createProtocol(protocolCfg);
                if (!protocol) {
                    throw std::runtime_error("Failed to create protocol: " + protocolCfg.id);
                }
                newProtocols.push_back(std::move(protocol));
            }

            // 5. Critical section: swap in new configuration
            {
                std::lock_guard<std::mutex> lock(m_configMutex);

                // Update routes first (will be used by new messages)
                updateRoutes(config.routes);

                // Then swap drivers and protocols
                swapDrivers(std::move(newDrivers));
                swapProtocols(std::move(newProtocols));

                // Update topology paths
                updateTopologyPaths(config.topologyPaths);

                // Store new configuration
                m_currentConfig = config;
            }

            // 6. Shutdown old drivers and protocols
            shutdownOldComponents();

            return Result::Success;

        } catch (const std::exception& e) {
            // Rollback on any error
            logError("Configuration rollback: " + std::string(e.what()));

            try {
                // Restore previous configuration
                m_currentConfig = backup;

                // Recreate previous drivers, protocols, routes
                restoreFromBackup(backup);
            } catch (...) {
                // Critical failure during rollback
                logError("Critical failure during rollback");
                return Result::ConfigurationInvalid;
            }

            return Result::ConfigurationInvalid;
        }
    }

private:
    SystemConfig m_currentConfig;
    std::mutex m_configMutex;
};
```

### 2.2 OpcodeRegistry Best Practices

The OpcodeRegistry is responsible for managing OPCODE definitions and validations:

```cpp
class BestOpcodeRegistry {
public:
    // Initialize with standard system OPCODEs
    void initializeSystemOpcodes() {
        // Register standard system OPCODEs
        registerSystemOpcode(0x0001, "SYS_HEARTBEAT", 16, Priority::LOW, AckPolicy::None);
        registerSystemOpcode(0x0002, "SYS_POWER_STATE", 8, Priority::HIGH, AckPolicy::RequireAck);
        registerSystemOpcode(0x0003, "SYS_ERROR", 128, Priority::URGENT, AckPolicy::RequireAck);
        registerSystemOpcode(0x0004, "SYS_CONFIG_UPDATE", 256, Priority::HIGH, AckPolicy::RequireAck);

        // Load from configuration file
        loadOpcodeDefinitions("opcode_definitions.json");
    }

    // Check OPCODE format and contents during registration
    Result registerOpcode(const OpcodeInfo& info) {
        // 1. Check if already registered
        if (m_opcodes.count(info.opcode) > 0) {
            if (!overwriteAllowed(info.opcode)) {
                logError("OPCODE already registered: 0x" + toHexString(info.opcode));
                return Result::InvalidOpcode;
            }
        }

        // 2. Validate OPCODE range
        if (!isValidOpcodeRange(info.opcode, info.name)) {
            logError("OPCODE out of valid range: 0x" + toHexString(info.opcode));
            return Result::InvalidOpcode;
        }

        // 3. Validate payload size
        if (info.maxPayloadSize > MAX_ALLOWED_PAYLOAD) {
            logError("Max payload size exceeds limit for OPCODE: 0x" + toHexString(info.opcode));
            return Result::InvalidOpcode;
        }

        // 4. Check for restricted names
        if (isReservedName(info.name)) {
            logError("Reserved OPCODE name: " + info.name);
            return Result::InvalidOpcode;
        }

        // 5. Register OPCODE
        m_opcodes[info.opcode] = info;
        m_opcodesByName[info.name] = info.opcode;

        // 6. Notify subscribers of registry changes
        if (m_registryUpdateCallback) {
            m_registryUpdateCallback(info);
        }

        return Result::Success;
    }

    // Optimize lookup with fast path for common OPCODEs
    bool validateOpcodeAndGetInfo(uint16_t opcode, OpcodeInfo& outInfo) {
        // Fast path: check common OPCODEs first (small cache)
        if (auto it = m_commonOpcodeCache.find(opcode); it != m_commonOpcodeCache.end()) {
            outInfo = it->second;
            return true;
        }

        // Normal lookup
        auto it = m_opcodes.find(opcode);
        if (it == m_opcodes.end()) {
            return false;
        }

        outInfo = it->second;

        // Update cache if this is becoming a common OPCODE
        updateUsageStatistics(opcode);
        if (isCommonOpcode(opcode)) {
            m_commonOpcodeCache[opcode] = outInfo;
        }

        return true;
    }

    // Support for upgrading/deprecating OPCODEs
    void deprecateOpcode(uint16_t opcode, uint16_t replacementOpcode) {
        auto it = m_opcodes.find(opcode);
        if (it != m_opcodes.end()) {
            it->second.deprecated = true;
            if (replacementOpcode != 0) {
                it->second.description += " [DEPRECATED: Use 0x" +
                    toHexString(replacementOpcode) + " instead]";
            }
        }
    }

private:
    std::unordered_map<uint16_t, OpcodeInfo> m_opcodes;
    std::unordered_map<std::string, uint16_t> m_opcodesByName;
    std::unordered_map<uint16_t, OpcodeInfo> m_commonOpcodeCache;
    std::function<void(const OpcodeInfo&)> m_registryUpdateCallback;

    // Track usage for cache optimization
    std::unordered_map<uint16_t, uint32_t> m_usageCount;
};
```

### 2.3 RouteManager Best Practices

The RouteManager handles routing decisions and path resolution:

```cpp
class BestRouteManager : public IRouter {
public:
    // Efficient subscriber lookup with path caching
    std::vector<ForwardingTarget> resolveOpcodeTargets(uint16_t opcode) override {
        // 1. Check cache first for common OPCODEs
        if (auto it = m_targetCache.find(opcode); it != m_targetCache.end()) {
            // Return cached result if still valid
            if (!isCacheExpired(it->second)) {
                return it->second.targets;
            }
        }

        // 2. Find all subscribers for this OPCODE
        std::vector<ForwardingTarget> targets;

        // Fast path: direct subscriptions on this node
        auto localSubs = m_localSubscriptions.find(opcode);
        if (localSubs != m_localSubscriptions.end()) {
            ForwardingTarget localTarget;
            localTarget.isLocal = true;
            localTarget.subscriptionIndex = localSubs->second;
            targets.push_back(localTarget);
        }

        // Find remote subscriptions that need routing
        auto remoteSubs = m_remoteSubscriptions.find(opcode);
        if (remoteSubs != m_remoteSubscriptions.end()) {
            for (const auto& destNodeId : remoteSubs->second) {
                // Find best path to destination
                auto path = findBestPath(m_nodeId, destNodeId);
                if (path.isEmpty()) {
                    logWarning("No path found to node: " + destNodeId);
                    continue;
                }

                // Create forwarding target with next hop info
                ForwardingTarget target;
                target.isLocal = false;
                target.nextHop = path.firstHop();
                target.driverId = path.firstDriverId();
                target.protocolId = path.protocolId;
                target.finalDestination = destNodeId;
                target.remainingHops = path.remainingHops();

                targets.push_back(target);
            }
        }

        // 3. Update cache for future lookups
        if (!targets.empty()) {
            CacheEntry entry;
            entry.targets = targets;
            entry.timestamp = OsClock::nowMicros();
            entry.expiryTime = entry.timestamp + CACHE_LIFETIME_US;
            m_targetCache[opcode] = entry;
        }

        return targets;
    }

    // Add a new route with constraint validation
    Result addRoute(const RouteConfig& route) override {
        // 1. Validate route configuration
        if (route.id.empty() || route.src.empty() || route.dst.empty() ||
            route.driverId.empty() || route.protocolId.empty()) {
            logError("Invalid route configuration: missing required fields");
            return Result::ConfigurationInvalid;
        }

        // 2. Check if driver exists and meets capabilities
        auto driver = m_driverManager->getDriver(route.driverId);
        if (!driver) {
            logError("Route references non-existent driver: " + route.driverId);
            return Result::ConfigurationInvalid;
        }

        // 3. Validate capability requirements
        auto driverCaps = driver->getCapabilities();
        for (const auto& reqCap : route.requiresCaps) {
            if (!evaluateCapabilityConstraint(driverCaps, reqCap)) {
                logError("Driver " + route.driverId + " does not meet requirement: " + reqCap);
                return Result::ConfigurationInvalid;
            }
        }

        // 4. Check for duplicate route ID
        if (m_routes.find(route.id) != m_routes.end()) {
            logWarning("Replacing existing route: " + route.id);
        }

        // 5. Store route
        {
            std::lock_guard<std::mutex> lock(m_routeMutex);
            m_routes[route.id] = route;

            // 6. Update route graphs
            updateRouteGraphs(route);

            // 7. Invalidate affected path caches
            invalidatePathCache(route.src, route.dst);
        }

        return Result::Success;
    }

    // Add a multi-hop path
    Result addPath(const TopologyPath& path) override {
        // 1. Validate path
        if (path.destination.empty() || path.hops.empty()) {
            logError("Invalid path: missing destination or hops");
            return Result::ConfigurationInvalid;
        }

        // 2. Verify all referenced routes exist
        for (const auto& hopId : path.hops) {
            if (m_routes.find(hopId) == m_routes.end()) {
                logError("Path references non-existent route: " + hopId);
                return Result::ConfigurationInvalid;
            }
        }

        // 3. Check for route continuity (each hop connects to the next)
        if (!validatePathContinuity(path)) {
            logError("Path has discontinuities: " + path.destination);
            return Result::ConfigurationInvalid;
        }

        // 4. Store path
        {
            std::lock_guard<std::mutex> lock(m_routeMutex);
            m_paths[path.destination] = path;

            // 5. Pre-compute and cache path details for faster lookup
            precomputePath(path);
        }

        return Result::Success;
    }

    // Rebuild routing tables after configuration changes
    void rebuildRoutingTables() {
        std::lock_guard<std::mutex> lock(m_routeMutex);

        // 1. Clear existing tables and caches
        m_routeGraph.clear();
        m_pathCache.clear();
        m_targetCache.clear();

        // 2. Rebuild route graph from routes
        for (const auto& [id, route] : m_routes) {
            updateRouteGraphs(route);
        }

        // 3. Pre-compute common paths
        for (const auto& [dest, path] : m_paths) {
            precomputePath(path);
        }

        // 4. Run Dijkstra's algorithm to find shortest paths between all nodes
        computeAllPairShortestPaths();
    }

    // Register an OPCODE subscription for routing purposes
    void registerSubscription(uint16_t opcode, const std::string& nodeId) {
        std::lock_guard<std::mutex> lock(m_routeMutex);

        if (nodeId == m_nodeId) {
            // Local subscription
            m_localSubscriptions[opcode].push_back(m_nextSubscriptionIndex++);
        } else {
            // Remote subscription
            m_remoteSubscriptions[opcode].insert(nodeId);
        }

        // Invalidate target cache for this OPCODE
        m_targetCache.erase(opcode);
    }

    // Find best path between nodes using Dijkstra's algorithm
    Path findBestPath(const std::string& src, const std::string& dst) {
        // 1. Check path cache first
        auto cacheKey = src + "->" + dst;
        if (auto it = m_pathCache.find(cacheKey); it != m_pathCache.end()) {
            return it->second;
        }

        // 2. Run Dijkstra's algorithm if not in cache
        Path path = runDijkstraAlgorithm(src, dst);

        // 3. Cache result for future lookups
        if (!path.isEmpty()) {
            m_pathCache[cacheKey] = path;
        }

        return path;
    }

    // Handle topology changes (e.g., node/link failures)
    void handleTopologyChange(const std::string& nodeId, bool isAvailable) {
        std::lock_guard<std::mutex> lock(m_routeMutex);

        // 1. Mark node as available/unavailable
        if (isAvailable) {
            m_unavailableNodes.erase(nodeId);
        } else {
            m_unavailableNodes.insert(nodeId);
        }

        // 2. Clear affected path caches
        clearPathCachesForNode(nodeId);

        // 3. Recompute paths if needed
        if (!isAvailable) {
            // Need to find alternative paths immediately
            computeAllPairShortestPaths();
        }
    }

private:
    // Internal data structures
    struct CacheEntry {
        std::vector<ForwardingTarget> targets;
        uint64_t timestamp;
        uint64_t expiryTime;
    };

    struct Path {
        std::vector<std::string> hops;
        std::vector<std::string> driverIds;
        std::string protocolId;
        uint32_t totalCost;

        bool isEmpty() const { return hops.empty(); }
        std::string firstHop() const { return hops.empty() ? "" : hops[0]; }
        std::string firstDriverId() const { return driverIds.empty() ? "" : driverIds[0]; }
        std::vector<std::string> remainingHops() const {
            return hops.size() <= 1 ? std::vector<std::string>{} :
                std::vector<std::string>(hops.begin() + 1, hops.end());
        }
    };

    // RouteGraph for Dijkstra's algorithm
    struct RouteGraphNode {
        std::map<std::string, std::pair<uint32_t, std::string>> neighbors; // node -> (cost, routeId)
    };

    // Member variables
    std::string m_nodeId;
    std::map<std::string, RouteConfig> m_routes;
    std::map<std::string, TopologyPath> m_paths;
    std::map<std::string, RouteGraphNode> m_routeGraph;
    std::map<std::string, Path> m_pathCache;
    std::unordered_map<uint16_t, CacheEntry> m_targetCache;
    std::unordered_map<uint16_t, std::vector<size_t>> m_localSubscriptions;
    std::unordered_map<uint16_t, std::set<std::string>> m_remoteSubscriptions;
    std::set<std::string> m_unavailableNodes;
    std::mutex m_routeMutex;
    size_t m_nextSubscriptionIndex = 0;
    IDriverManager* m_driverManager = nullptr;

    static constexpr uint64_t CACHE_LIFETIME_US = 60'000'000; // 60 seconds

    // Private helper methods
    void updateRouteGraphs(const RouteConfig& route) {
        // Add bidirectional edge to route graph
        uint32_t cost = computeRouteCost(route);
        m_routeGraph[route.src].neighbors[route.dst] = {cost, route.id};

        // If route is bidirectional, add reverse edge too
        if (route.bidirectional) {
            m_routeGraph[route.dst].neighbors[route.src] = {cost, route.id};
        }
    }

    bool validatePathContinuity(const TopologyPath& path) {
        std::string currentNode;

        // Get starting node from first hop
        if (!path.hops.empty()) {
            const auto& firstHopId = path.hops[0];
            auto it = m_routes.find(firstHopId);
            if (it != m_routes.end()) {
                currentNode = it->second.src;
            }
        }

        // Check each hop connects to the next
        for (const auto& hopId : path.hops) {
            auto it = m_routes.find(hopId);
            if (it == m_routes.end()) {
                return false;
            }

            const auto& route = it->second;

            // Check this route starts where we currently are
            if (route.src != currentNode) {
                return false;
            }

            // Move to next node
            currentNode = route.dst;
        }

        // Final node should be the destination
        return currentNode == path.destination;
    }

    uint32_t computeRouteCost(const RouteConfig& route) {
        // Base cost from priority
        uint32_t cost = 10;
        switch (route.priority) {
            case Priority::URGENT: cost = 1; break;
            case Priority::HIGH: cost = 5; break;
            case Priority::NORMAL: cost = 10; break;
            case Priority::LOW: cost = 20; break;
        }

        // Adjust based on driver capabilities
        auto driver = m_driverManager->getDriver(route.driverId);
        if (driver) {
            auto caps = driver->getCapabilities();

            // Add latency-based cost if available
            if (caps.contains("latency")) {
                try {
                    int latency = std::stoi(caps.at("latency"));
                    cost += latency / 10; // Scale latency contribution
                } catch (...) {
                    // Ignore conversion errors
                }
            }

            // Prefer DMA-capable drivers
            if (caps.contains("dma") && caps.at("dma") == "true") {
                cost -= 2; // Lower cost for DMA capable
            }

            // Prefer higher bandwidth
            if (caps.contains("bandwidth")) {
                try {
                    int bandwidth = std::stoi(caps.at("bandwidth"));
                    cost -= std::min(5, bandwidth / 1000); // Max 5 reduction
                } catch (...) {
                    // Ignore conversion errors
                }
            }
        }

        return std::max(1u, cost); // Ensure minimum cost of 1
    }

    void precomputePath(const TopologyPath& path) {
        // Extract source from first hop
        if (path.hops.empty()) return;

        const auto& firstHopId = path.hops[0];
        auto routeIt = m_routes.find(firstHopId);
        if (routeIt == m_routes.end()) return;

        std::string src = routeIt->second.src;

        // Create path object
        Path computedPath;
        computedPath.totalCost = path.totalCost;

        // Process each hop to build the complete path
        std::string currentNode = src;
        for (const auto& hopId : path.hops) {
            auto it = m_routes.find(hopId);
            if (it == m_routes.end()) continue;

            const auto& route = it->second;
            computedPath.hops.push_back(route.dst);
            computedPath.driverIds.push_back(route.driverId);

            // Use protocol from first hop
            if (computedPath.protocolId.empty()) {
                computedPath.protocolId = route.protocolId;
            }

            currentNode = route.dst;
        }

        // Cache the computed path
        std::string cacheKey = src + "->" + path.destination;
        m_pathCache[cacheKey] = computedPath;
    }

    void invalidatePathCache(const std::string& src, const std::string& dst) {
        // Remove direct path
        m_pathCache.erase(src + "->" + dst);

        // Remove any paths that might use this link
        // (conservative approach: clear all paths that start from src or end at dst)
        for (auto it = m_pathCache.begin(); it != m_pathCache.end();) {
            const auto& key = it->first;
            if (key.starts_with(src + "->") || key.ends_with("->" + dst)) {
                it = m_pathCache.erase(it);
            } else {
                ++it;
            }
        }
    }

    void clearPathCachesForNode(const std::string& nodeId) {
        // Clear all paths that involve this node
        for (auto it = m_pathCache.begin(); it != m_pathCache.end();) {
            const auto& key = it->first;
            if (key.starts_with(nodeId + "->") || key.ends_with("->" + nodeId)) {
                it = m_pathCache.erase(it);
            } else {
                ++it;
            }
        }
    }

    bool isCacheExpired(const CacheEntry& entry) {
        return OsClock::nowMicros() > entry.expiryTime;
    }

    // Dijkstra's algorithm implementation for finding shortest paths
    Path runDijkstraAlgorithm(const std::string& src, const std::string& dst) {
        // Check if source or destination are unavailable
        if (m_unavailableNodes.find(src) != m_unavailableNodes.end() ||
            m_unavailableNodes.find(dst) != m_unavailableNodes.end()) {
            return Path{}; // Empty path - no route available
        }

        // Priority queue for Dijkstra's algorithm
        // pair<cost, node>
        std::priority_queue<
            std::pair<uint32_t, std::string>,
            std::vector<std::pair<uint32_t, std::string>>,
            std::greater<std::pair<uint32_t, std::string>>
        > pq;

        // Distance map
        std::map<std::string, uint32_t> distance;

        // Previous node map for path reconstruction
        std::map<std::string, std::string> previous;

        // Route ID used for each hop
        std::map<std::string, std::string> routeIds;

        // Initialize distances to infinity
        for (const auto& [node, _] : m_routeGraph) {
            distance[node] = std::numeric_limits<uint32_t>::max();
        }

        // Source distance is 0
        distance[src] = 0;
        pq.push({0, src});

        // Dijkstra's algorithm
        while (!pq.empty()) {
            auto [currentDist, currentNode] = pq.top();
            pq.pop();

            // Skip if we've found a better path already
            if (currentDist > distance[currentNode]) {
                continue;
            }

            // If we reached destination, we're done
            if (currentNode == dst) {
                break;
            }

            // Skip unavailable nodes
            if (m_unavailableNodes.find(currentNode) != m_unavailableNodes.end()) {
                continue;
            }

            // Check all neighbors
            if (auto it = m_routeGraph.find(currentNode); it != m_routeGraph.end()) {
                for (const auto& [neighbor, edgeInfo] : it->second.neighbors) {
                    auto [edgeCost, routeId] = edgeInfo;

                    // Skip unavailable neighbors
                    if (m_unavailableNodes.find(neighbor) != m_unavailableNodes.end()) {
                        continue;
                    }

                    uint32_t newDist = currentDist + edgeCost;

                    // If we found a better path, update
                    if (newDist < distance[neighbor]) {
                        distance[neighbor] = newDist;
                        previous[neighbor] = currentNode;
                        routeIds[neighbor] = routeId;
                        pq.push({newDist, neighbor});
                    }
                }
            }
        }

        // Check if destination is reachable
        if (distance[dst] == std::numeric_limits<uint32_t>::max()) {
            return Path{}; // No path found
        }

        // Reconstruct path
        Path path;
        path.totalCost = distance[dst];

        std::string current = dst;
        std::vector<std::string> reversedHops;
        std::vector<std::string> reversedDriverIds;
        std::string firstRouteId;

        while (current != src) {
            std::string prev = previous[current];
            std::string routeId = routeIds[current];

            // Store first route ID for protocol
            if (firstRouteId.empty()) {
                firstRouteId = routeId;
            }

            // Add hop
            reversedHops.push_back(current);

            // Get driver ID from route
            auto routeIt = m_routes.find(routeId);
            if (routeIt != m_routes.end()) {
                reversedDriverIds.push_back(routeIt->second.driverId);
            }

            current = prev;
        }

        // Reverse to get correct order
        std::reverse(reversedHops.begin(), reversedHops.end());
        std::reverse(reversedDriverIds.begin(), reversedDriverIds.end());

        path.hops = reversedHops;
        path.driverIds = reversedDriverIds;

        // Get protocol from first route
        if (!firstRouteId.empty()) {
            auto routeIt = m_routes.find(firstRouteId);
            if (routeIt != m_routes.end()) {
                path.protocolId = routeIt->second.protocolId;
            }
        }

        return path;
    }

    // Floyd-Warshall algorithm for all-pairs shortest paths
    void computeAllPairShortestPaths() {
        // Get all nodes
        std::vector<std::string> nodes;
        for (const auto& [node, _] : m_routeGraph) {
            if (m_unavailableNodes.find(node) == m_unavailableNodes.end()) {
                nodes.push_back(node);
            }
        }

        // Initialize distance matrix
        const uint32_t INF = std::numeric_limits<uint32_t>::max();
        size_t n = nodes.size();
        std::vector<std::vector<uint32_t>> dist(n, std::vector<uint32_t>(n, INF));
        std::vector<std::vector<int>> next(n, std::vector<int>(n, -1));

        // Map node string to index
        std::map<std::string, size_t> nodeIndex;
        for (size_t i = 0; i < n; ++i) {
            nodeIndex[nodes[i]] = i;
            dist[i][i] = 0; // Distance to self is 0
        }

        // Initialize distances from route graph
        for (size_t i = 0; i < n; ++i) {
            const auto& nodeStr = nodes[i];
            if (auto it = m_routeGraph.find(nodeStr); it != m_routeGraph.end()) {
                for (const auto& [neighbor, edgeInfo] : it->second.neighbors) {
                    // Skip unavailable neighbors
                    if (m_unavailableNodes.find(neighbor) != m_unavailableNodes.end()) {
                        continue;
                    }

                    if (auto idx = nodeIndex.find(neighbor); idx != nodeIndex.end()) {
                        size_t j = idx->second;
                        dist[i][j] = edgeInfo.first; // Edge cost
                        next[i][j] = j; // Direct connection
                    }
                }
            }
        }

        // Floyd-Warshall algorithm
        for (size_t k = 0; k < n; ++k) {
            for (size_t i = 0; i < n; ++i) {
                for (size_t j = 0; j < n; ++j) {
                    if (dist[i][k] != INF && dist[k][j] != INF &&
                        dist[i][k] + dist[k][j] < dist[i][j]) {
                        dist[i][j] = dist[i][k] + dist[k][j];
                        next[i][j] = next[i][k];
                    }
                }
            }
        }

        // Reconstruct and cache all paths
        for (size_t i = 0; i < n; ++i) {
            for (size_t j = 0; j < n; ++j) {
                if (i != j && dist[i][j] != INF) {
                    // Reconstruct path from i to j
                    Path path;
                    path.totalCost = dist[i][j];

                    // Build path using next matrix
                    size_t curr = i;
                    while (curr != j) {
                        size_t nextNode = next[curr][j];
                        if (nextNode == -1) break; // No path

                        path.hops.push_back(nodes[nextNode]);

                        // Find route between curr and nextNode
                        std::string routeId = findRouteId(nodes[curr], nodes[nextNode]);
                        if (!routeId.empty()) {
                            auto routeIt = m_routes.find(routeId);
                            if (routeIt != m_routes.end()) {
                                path.driverIds.push_back(routeIt->second.driverId);

                                // Use protocol from first hop
                                if (path.protocolId.empty()) {
                                    path.protocolId = routeIt->second.protocolId;
                                }
                            }
                        }

                        curr = nextNode;
                    }

                    // Cache the path
                    std::string cacheKey = nodes[i] + "->" + nodes[j];
                    m_pathCache[cacheKey] = path;
                }
            }
        }
    }

    std::string findRouteId(const std::string& src, const std::string& dst) {
        if (auto it = m_routeGraph.find(src); it != m_routeGraph.end()) {
            if (auto nit = it->second.neighbors.find(dst); nit != it->second.neighbors.end()) {
                return nit->second.second; // Return route ID
            }
        }
        return "";
    }

    // Evaluate capability constraints like "maxFrame>=256", "latency<100"
    bool evaluateCapabilityConstraint(const std::map<std::string, std::string>& caps,
                                      const std::string& constraint) {
        // Parse constraint: <key><operator><value>
        // Operators: ==, !=, >, <, >=, <=

        // Find operator
        size_t opPos = std::string::npos;
        std::string op;

        // Check for two-character operators first
        for (const auto& opStr : {"==", "!=", ">=", "<="}) {
            opPos = constraint.find(opStr);
            if (opPos != std::string::npos) {
                op = opStr;
                break;
            }
        }

        // If not found, check single-character operators
        if (opPos == std::string::npos) {
            for (char opChar : {'>', '<'}) {
                opPos = constraint.find(opChar);
                if (opPos != std::npos) {
                    op = opChar;
                    break;
                }
            }
        }

        // If no operator found, invalid constraint
        if (opPos == std::string::npos) {
            logError("Invalid capability constraint: " + constraint);
            return false;
        }

        // Extract key and value
        std::string key = constraint.substr(0, opPos);
        std::string valueStr = constraint.substr(opPos + op.length());

        // Check if capability exists
        if (caps.find(key) == caps.end()) {
            // Special case: if constraint is "key==false" and key doesn't exist, it's true
            return (op == "==" and valueStr == "false")
        }

        # Get actual value
        std::string actualStr = caps.at(key);

        # Compare based on operator
        if (op == "==") {
            return actualStr == valueStr;
        } else if (op == "!=") {
            return actualStr != valueStr;
        }

        // For other operators, try numeric comparison
        try {
            int actual = std::stoi(actualStr);
            int expected = std::stoi(valueStr);

            if (op == ">") return actual > expected;
            if (op == "<") return actual < expected;
            if (op == ">=") return actual >= expected;
            if (op == "<=") return actual <= expected;
        } catch (...) {
            logError("Non-numeric comparison: " + constraint);
            return false;
        }

        return false;
    }
};
```

The extended RouteManager provides sophisticated path finding and routing capabilities:

1. **Efficient Path Resolution**:
   - Caches common routes for performance
   - Implements Dijkstra's algorithm for optimal path finding
   - Supports multi-hop routing across complex topologies

2. **Dynamic Routing**:
   - Handles node and link failures gracefully
   - Recomputes paths when topology changes
   - Supports primary and backup paths

3. **Path Optimization**:
   - Considers multiple factors in cost calculation (latency, bandwidth, DMA)
   - Applies QoS requirements to path selection
   - Validates capability constraints for routes

4. **Advanced Algorithms**:
   - Dijkstra's algorithm for single-source shortest paths
   - Floyd-Warshall algorithm for all-pairs shortest paths
   - Path reconstruction and validation

This implementation ensures messages are routed optimally across the distributed system, adapting to changing conditions while maintaining high performance through strategic caching.
