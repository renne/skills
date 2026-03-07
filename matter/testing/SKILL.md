---
name: testing
description: Matter SDK testing including unit tests with pw_unit_test (GoogleTest), Python integration tests with Mobly and ChipDeviceCtrl, YAML tests, CI test configuration, and running tests locally with chip-tool or the Python controller. Use when writing Matter device tests, running certification tests, setting up CI for Matter tests, or debugging test failures in the Matter SDK.
---
# Matter SDK Testing

Source: https://project-chip.github.io/connectedhomeip-doc/testing/index.html

## Overview

Matter has two main testing layers:

| Layer | Tools | Use Case |
|-------|-------|----------|
| **Unit tests** | `pw_unit_test` (GoogleTest) | Fast, isolated cluster/component logic tests |
| **Integration tests** | YAML + chip-tool, Python + Mobly | Full-stack tests, certification testing |

---

## Unit Tests

### Why Unit Test?

- Much faster than integration tests
- Run as part of the build process
- Can test specific error conditions (OOM, invalid state) that are hard to trigger otherwise
- Test feature combinations not in example apps
- Enables testing without hardware

### Framework: `pw_unit_test` (GoogleTest)

Matter uses [Pigweed's pw_unit_test](https://pigweed.dev/pw_unit_test/), which is a GoogleTest implementation.

```cpp
#include <pw_unit_test/framework.h>

TEST(MyTestSuite, BasicTest)
{
    SomeClass obj;
    obj.DoSomething();
    EXPECT_EQ(obj.GetResult(), 42);

    SomePointer * ptr = obj.GetPtr();
    ASSERT_NE(ptr, nullptr);  // Aborts test if null
    ptr->DoMore();
}
```

### Test Fixtures (Setup/Teardown)

```cpp
#include <pw_unit_test/framework.h>

class MyTestContext : public ::testing::Test
{
public:
    static void SetUpTestSuite()    // Once per suite
    {
        sFixture.Init();
        ASSERT_TRUE(sFixture.WorkingGreat());
    }
    static void TearDownTestSuite() // Once per suite
    {
        sFixture.Shutdown();
    }

protected:
    void SetUp() override           // Once per test
    {
        mPerTest.Init();
        ASSERT_TRUE(mPerTest.WorkingGreat());
    }
    void TearDown() override        // Once per test
    {
        mPerTest.Shutdown();
    }

private:
    static SomeFixture sFixture;
    AnotherFixture mPerTest;
};
SomeFixture MyTestContext::sFixture;

TEST_F(MyTestContext, MySpecificTest)
{
    mPerTest.DoSomething();
    EXPECT_EQ(mPerTest.GetResult(), 7);
}
```

### Loopback Messaging Context

For tests requiring network messaging:

```cpp
#include <app/tests/AppTestContext.h>

// Simple alias if no customization needed
using MyTestContext = Test::AppContext;

// Or derive for custom setup
class MyTestContext : public Test::AppContext
{
public:
    static void SetUpTestSuite()
    {
        AppContext::SetUpTestSuite();
        VerifyOrReturn(!HasFailure());
        // custom setup...
    }
protected:
    void SetUp() override
    {
        AppContext::SetUp();
        VerifyOrReturn(!HasFailure());
        // custom per-test setup...
    }
};

TEST_F(MyTestContext, TestWithMessaging)
{
    // Default controller: secure session over loopback
    // ...
}
```

### Assertion Best Practices

```cpp
// Preferred: specific assertions
EXPECT_EQ(result, 3);
EXPECT_GT(result, 1);
EXPECT_STREQ(myString, "hello");

// Avoid: generic boolean assertions
EXPECT_TRUE(result == 3);   // less informative on failure
EXPECT_TRUE(result > 1);

// Use ASSERT_* to abort the test on failure
ASSERT_NE(ptr, nullptr);    // won't proceed if null
ptr->Method();

// Check after subroutine with ASSERT
Subroutine();
VerifyOrReturn(!HasFailure());  // stop if subroutine asserted

// Force failure
FAIL();         // fatal
ADD_FAILURE();  // non-fatal
```

### Available Test Utilities

| Utility | Description |
|---------|-------------|
| `MockClock` | Mock `System::Clock` for time-dependent tests |
| `TestPersistentStorageDelegate` | In-memory storage for tests needing persistence |
| `Test::AppContext` | Loopback network + two secure sessions |

### Compiling and Running Unit Tests

```bash
# Build all tests for host
gn gen out/host && ninja -C out/host check

# Build and run a specific test
ninja -C out/host src/app/tests:tests_run

# Build individual test binary
ninja -C out/debug linux_x64_clang/tests/TestAccessControl
```

---

## Integration Tests

Integration tests use a controller and one or more device apps to test the full stack.

### YAML Tests

YAML tests are human-readable test definitions run through the chip-tool controller.

**Pros:** Simple to write, easy for certification ATLs to read
**Cons:** Limited conditionals, no branch control

```yaml
name: Test Basic Information
config:
    nodeId: 0x12344321
    cluster: Basic Information
    endpoint: 0

tests:
    - label: "Read VendorName"
      command: readAttribute
      attribute: VendorName
      response:
          value: "Test Vendor"
          constraints:
              type: char_string

    - label: "Verify ClusterRevision"
      command: readAttribute
      attribute: ClusterRevision
      response:
          value: 3
```

### Python Tests (Recommended for Complex Tests)

Python tests use the [Mobly](https://github.com/google/mobly) framework and `ChipDeviceCtrl`.

**Pros:** Full Python, complete control API, easier to extend
**Cons:** More verbose, steeper learning curve

#### Minimal Python Test

```python
# === BEGIN CI TEST ARGUMENTS ===
# test-runner-runs:
#   run1:
#     app: ${ALL_CLUSTERS_APP}
#     app-args: --discriminator 1234 --KVS kvs1
#     script-args: >
#       --storage-path admin_storage.json
#       --commissioning-method on-network
#       --discriminator 1234
#       --passcode 20202021
#     factory-reset: true
#     quiet: true
# === END CI TEST ARGUMENTS ===

import matter.clusters as Clusters
from matter.testing.matter_testing import MatterBaseTest, async_test_body, default_matter_test_main
from mobly import asserts

class TC_BINFO_1_1(MatterBaseTest):

    @async_test_body
    async def test_TC_BINFO_1_1(self):
        vendor_name = await self.read_single_attribute_check_success(
            cluster=Clusters.BasicInformation,
            attribute=Clusters.BasicInformation.Attributes.VendorName,
            endpoint=0,
        )
        asserts.assert_equal(vendor_name, "Test vendor name", "Unexpected vendor name")

if __name__ == "__main__":
    default_matter_test_main()
```

Test method naming for certification:  `test_TC_PICSCODE_#_#` (e.g., `test_TC_BINFO_1_1`)

Test files go in `src/python_testing/`.

### ChipDeviceCtrl API (Python Controller)

```python
# Reading attributes
result = await dev_ctrl.ReadAttribute(
    node_id,
    [(endpoint, Clusters.OnOff.Attributes.OnOff)]
)

# Writing attributes
await dev_ctrl.WriteAttribute(
    node_id,
    [(endpoint, Clusters.OnOff.Attributes.OnTime(value=10))]
)

# Invoking commands
await dev_ctrl.SendCommand(
    node_id, endpoint,
    Clusters.OnOff.Commands.Toggle()
)

# Subscribing
sub = await dev_ctrl.ReadAttribute(
    node_id,
    [(endpoint, Clusters.OnOff.Attributes.OnOff)],
    reportInterval=(0, 30),
    keepSubscriptions=True
)
```

### Python Cluster Codegen Reference

```python
import matter.clusters as Clusters

# Access cluster elements
cluster_id = Clusters.OnOff.id
attr_class = Clusters.OnOff.Attributes.OnTime
cmd = Clusters.OnOff.Commands.OnWithTimedOff(onOffControl=0, onTime=5, offWaitTime=8)
privilege = Clusters.AccessControl.Enums.AccessControlEntryPrivilegeEnum.kAdminister
feature = Clusters.LaundryWasherControls.Bitmaps.Feature.kSpin

# Structs
appearance = Clusters.BasicInformation.Structs.ProductAppearanceStruct(
    finish=Clusters.BasicInformation.Enums.ProductFinishEnum.kFabric,
    primaryColor=Clusters.BasicInformation.Enums.ColorEnum.kBlack
)

# Lookup by ID
from matter.clusters import ClusterObjects
cluster = ClusterObjects.ALL_CLUSTERS[cluster_id]
attr = ClusterObjects.ALL_ATTRIBUTES[cluster_id][attribute_id]
```

---

## Running Integration Tests Locally

### Basic Run (On-Network)

```bash
# Build chip-tool and the app
./scripts/build/build_examples.py \
    --target linux-x64-chip-tool \
    --target linux-x64-all-clusters build

# Run a specific YAML test
scripts/tests/run_test_suite.py --runner chip_tool_python \
    --target TestOperationalState \
    run \
    --app-path all-clusters:out/linux-x64-all-clusters/chip-all-clusters-app \
    --tool-path chip-tool:out/linux-x64-chip-tool/chip-tool
```

### Auto-discover Paths

```bash
scripts/tests/run_test_suite.py run --discover-paths --target TestOperationalState
```

### Run with Network Namespaces (Linux)

Enable on the host system:
```
# /etc/sysctl.conf or /etc/sysctl.d/99-matter.conf
kernel.unprivileged_userns_clone = 1
kernel.apparmor_restrict_unprivileged_userns = 0  # Ubuntu only
```

```bash
sudo sysctl --system
```

### Run with Mocked BLE+Wi-Fi (ble-wifi commissioning)

```bash
scripts/tests/run_test_suite.py --runner chip_tool_python \
    --target TestOperationalState \
    run \
    --app-path all-clusters:out/linux-x64-all-clusters/chip-all-clusters-app \
    --tool-path chip-tool:out/linux-x64-chip-tool/chip-tool \
    --commissioning-method ble-wifi
```

### Run Python Tests Directly

```bash
./scripts/tests/run_python_test.py \
    --app out/linux-x64-all-clusters/chip-all-clusters-app \
    --script src/python_testing/TC_BINFO_1_1.py
```

---

## PICS and PIXIT

- **PICS** (Protocol Implementation Conformance Statement) – Declares what features a device supports
- **PIXIT** (Protocol Implementation eXtra Information for Testing) – Extra parameters needed for testing

PICS/PIXIT files are used by the certification test harness to select applicable tests.

---

## CI Test Configuration

Python test CI arguments are defined in structured comments at the top of each test file:

```python
# === BEGIN CI TEST ARGUMENTS ===
# test-runner-runs:
#   run1:
#     app: ${ALL_CLUSTERS_APP}
#     app-args: --discriminator 1234 --KVS kvs1 --trace-to json:${TRACE_APP}.json
#     script-args: >
#       --storage-path admin_storage.json
#       --commissioning-method on-network
#       --discriminator 1234
#       --passcode 20202021
#       --trace-to json:${TRACE_TEST_JSON}.json
#       --trace-to perfetto:${TRACE_TEST_PERFETTO}.perfetto
#     factory-reset: true
#     quiet: true
# === END CI TEST ARGUMENTS ===
```

---

## References

- [Testing Guides Index](https://project-chip.github.io/connectedhomeip-doc/testing/index.html)
- [Integration and Certification Tests](https://project-chip.github.io/connectedhomeip-doc/testing/integration_tests.html)
- [Python Testing Framework](https://project-chip.github.io/connectedhomeip-doc/testing/python.html)
- [Unit Testing](https://project-chip.github.io/connectedhomeip-doc/testing/unit_testing.html)
- [Pigweed pw_unit_test](https://pigweed.dev/pw_unit_test/)
- [Mobly Test Framework](https://github.com/google/mobly)
- [ChipDeviceCtrl.py](https://github.com/project-chip/connectedhomeip/blob/master/src/controller/python/matter/ChipDeviceCtrl.py)
- [hello_test.py Example](https://github.com/project-chip/connectedhomeip/blob/master/src/python_testing/hello_test.py)
