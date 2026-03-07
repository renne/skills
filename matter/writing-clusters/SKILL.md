---
name: writing-clusters
description: Writing and implementing new Matter clusters in C++, including cluster XML definition, ServerClusterInterface, attribute/command/event handling, build system integration, and migration from Ember to code-driven clusters. Use when creating a new Matter cluster, implementing cluster logic in the SDK, or migrating an existing Ember-based cluster to the code-driven pattern.
---
# Writing and Updating Matter Clusters

Source: https://project-chip.github.io/connectedhomeip-doc/guides/writing_clusters.html

## Overview

A Matter cluster is a self-contained server-side object that implements a specific domain of device functionality (e.g., On/Off, Level Control, Door Lock). The SDK uses a **code-driven** approach where cluster logic is implemented in C++ classes implementing `ServerClusterInterface`.

### Key Stages

1. Define the cluster in XML (based on the Matter spec)
2. Implement the cluster in C++
3. Integrate with the build system
4. Connect to an application's code generation configuration
5. Write unit and integration tests

---

## Part 1: Cluster Definition (XML)

Clusters are defined in XML files in `src/app/zap-templates/zcl/data-model/chip/`.

- Use [Alchemy](https://github.com/project-chip/alchemy) to generate or update XML from the Matter spec's asciidoc. **Do not manually edit XML** — it is error-prone.
- After updating XML, run code generation:

```bash
./scripts/run_in_build_env.sh 'scripts/tools/zap_regen_all.py'
```

---

## Part 2: C++ Implementation

### File Structure

Create the cluster directory at `src/app/clusters/<cluster-directory>/`:

```
src/app/clusters/my-cluster-server/
├── BUILD.gn
├── app_config_dependent_sources.gni
├── app_config_dependent_sources.cmake
├── MyCluster.h
├── MyCluster.cpp
├── CodegenIntegration.cpp
└── tests/
    ├── BUILD.gn
    └── TestMyCluster.cpp
```

**Naming conventions:**
- Directory: `<cluster-name>-server` (e.g., `basic-information`)
- Header/source: `ClusterNameCluster.h` / `ClusterNameCluster.cpp`

Add directory mapping in `src/app/zap_cluster_list.json` under `ServerDirectories`:
```json
"MY_CLUSTER": "my-cluster-server"
```

### Recommended: Combined Implementation Pattern

For optimal flash and RAM usage, use a **combined implementation** where logic, data storage, and `ServerClusterInterface` are all in one class deriving from `DefaultServerCluster`:

```cpp
// MyCluster.h
#pragma once
#include <app/clusters/DefaultServerCluster.h>

namespace chip {
namespace app {
namespace Clusters {
namespace MyCluster {

class MyClusterServer : public DefaultServerCluster
{
public:
    MyClusterServer();

    // ServerClusterInterface methods
    CHIP_ERROR Startup(ServerClusterContext & context) override;
    void Shutdown() override;

    CHIP_ERROR ReadAttribute(const ConcreteReadAttributePath & path,
                             AttributeValueEncoder & encoder) override;

    CHIP_ERROR WriteAttribute(const ConcreteDataAttributePath & path,
                              AttributeValueDecoder & decoder) override;

    CHIP_ERROR InvokeCommand(const ConcreteCommandPath & path,
                             TLVReader * payload,
                             CommandHandler * handler) override;

private:
    uint16_t mMyAttribute = 0;
    ServerClusterContext * mContext = nullptr;
};

} // namespace MyCluster
} // namespace Clusters
} // namespace app
} // namespace chip
```

> **Note:** The legacy **Modular Implementation** (`ClusterLogic` + `ClusterImplementation`) is **discouraged** for new clusters due to added flash/RAM overhead.

### Design Principles

#### Handle Common Logic Internally

- Implement persistence (NVM), timers, and state machines within the cluster.
- The application should only be notified of significant events.

#### Delegate/Driver Pattern for Validation

When the application needs to participate in a cluster operation:

```cpp
class MyClusterDriver
{
public:
    virtual ~MyClusterDriver() = default;
    // Return Status::Success to allow, or another status to reject
    virtual Protocols::InteractionModel::Status
    OnMyAttributeChange(EndpointId endpoint, uint16_t newValue) = 0;
};
```

- Pre-write validation: driver can accept or reject before value is applied.
- Driver returns `Protocols::InteractionModel::Status`; non-Success rejects and propagates to initiator.
- Cluster performs spec-defined validations before calling driver.

#### Attribute and Feature Handling

```cpp
CHIP_ERROR ReadAttribute(const ConcreteReadAttributePath & path,
                         AttributeValueEncoder & encoder) override
{
    switch (path.mAttributeId)
    {
    case Attributes::MyAttribute::Id:
        return encoder.Encode(mMyAttribute);
    case Attributes::FeatureMap::Id:
        return encoder.Encode(mFeatureMap);
    case Attributes::ClusterRevision::Id:
        return encoder.Encode(kClusterRevision);
    default:
        return CHIP_ERROR_UNSUPPORTED_CHIP_FEATURE;
    }
}
```

#### Notifications After Writes

After a successful attribute write, always notify subscribers:

```cpp
CHIP_ERROR WriteAttribute(const ConcreteDataAttributePath & path,
                           AttributeValueDecoder & decoder) override
{
    switch (path.mAttributeId)
    {
    case Attributes::MyAttribute::Id: {
        uint16_t newValue;
        ReturnErrorOnFailure(decoder.Decode(newValue));
        if (mMyAttribute != newValue)
        {
            mMyAttribute = newValue;
            NotifyAttributeChangedIfSuccess(path);
        }
        return CHIP_NO_ERROR;
    }
    default:
        return CHIP_ERROR_UNSUPPORTED_CHIP_FEATURE;
    }
}
```

---

## Part 3: Build System Integration

### `BUILD.gn`

```gn
import("//build_overrides/build.gni")
import("//build_overrides/chip.gni")

source_set("my-cluster-server") {
    sources = [
        "MyCluster.h",
        "MyCluster.cpp",
    ]
    public_deps = [
        "${chip_root}/src/app:app",
        "${chip_root}/src/lib/core",
    ]
}
```

### `app_config_dependent_sources.gni`

```gn
app_config_dependent_sources = [ "CodegenIntegration.cpp" ]
```

### `app_config_dependent_sources.cmake`

```cmake
TARGET_SOURCES(
  ${APP_TARGET}
  PRIVATE
    "${CLUSTER_DIR}/CodegenIntegration.cpp"
    "${CLUSTER_DIR}/MyCluster.cpp"
    "${CLUSTER_DIR}/MyCluster.h"
)
```

---

## Part 4: Codegen Integration

`CodegenIntegration.cpp` connects the cluster to the application's code-generated configuration:

```cpp
// CodegenIntegration.cpp
#include "MyCluster.h"
#include <app/clusters/CodegenIntegrationHelpers.h>

namespace chip {
namespace app {
namespace Clusters {
namespace MyCluster {

// Register the cluster server with the application
CHIP_ERROR CodegenIntegration(MyClusterServer & server, EndpointId endpoint)
{
    return server.Startup(/* context */);
}

} // namespace MyCluster
} // namespace Clusters
} // namespace app
} // namespace chip
```

---

## Part 5: ZAP Configuration Updates

After implementing a code-driven cluster:

1. **Add to `CodeDrivenClusters`** in `src/app/common/templates/config-data.yaml`
2. **Remove from `CommandHandlerInterfaceOnlyClusters`** if present
3. **Add non-list attributes to `attributeAccessInterfaceAttributes`** in `src/app/zap-templates/zcl/zcl.json`
4. **Run codegen:**
   ```bash
   ./scripts/run_in_build_env.sh 'scripts/tools/zap_regen_all.py'
   ```

---

## Migrating Ember Clusters to Code-Driven

See the [migration guide](https://project-chip.github.io/connectedhomeip-doc/guides/migrating_ember_cluster_to_code_driven.html) for a step-by-step checklist. Key steps:

### PR Strategy (to simplify review)

1. **PR 1** – File renames only (e.g., `my-server.cpp` → `MyCluster.cpp`)
2. **PR 2** – Code movement only (reordering, anonymous namespace cleanup)
3. **PR 3** – Core logic changes (new class, attribute storage, command handlers)

### Implementation Mapping

| Ember Pattern | Code-Driven Equivalent |
|---------------|----------------------|
| `AttributeAccessInterface` (AAI) | `ReadAttribute` / `WriteAttribute` methods |
| `CommandHandlerInterface` (CHI) | `InvokeCommand` method |
| `emberAf<Cluster><Command>Callback` | `case` in `InvokeCommand` switch |
| `LogEvent(...)` | `mContext->interactionContext.eventsGenerator.GenerateEvent(...)` |
| `persist` attribute | Load in `Startup`, save on writes via `AttributePersistence` |
| `ram` attribute from ZAP | Load from generated `Accessors.h` in `CodegenIntegration.cpp` |

---

## References

- [Writing and Updating Clusters](https://project-chip.github.io/connectedhomeip-doc/guides/writing_clusters.html)
- [Migrating Ember Clusters to Code-Driven](https://project-chip.github.io/connectedhomeip-doc/guides/migrating_ember_cluster_to_code_driven.html)
- [Basic Information Cluster Example](https://github.com/project-chip/connectedhomeip/tree/master/src/app/clusters/basic-information)
- [src/app/zap_cluster_list.json](https://github.com/project-chip/connectedhomeip/blob/master/src/app/zap_cluster_list.json)
