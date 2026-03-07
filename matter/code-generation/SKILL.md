---
name: code-generation
description: Matter SDK code generation workflow using ZAP and the .matter IDL format, including editing .zap files, regenerating cluster code, using codegen.py, pre-generation for external apps, and the matter-data-model-xml-parser and matter-idl-lint tools. Use when adding or modifying clusters in a Matter application, regenerating code after spec changes, or working with Matter IDL files.
---
# Matter Code Generation

Source: https://project-chip.github.io/connectedhomeip-doc/zap_and_codegen/code_generation.html

## Overview

Matter relies on code generation to produce cluster-specific data types and callbacks. Code generation covers:

1. **Data serialization** – Structures, lists, and commands for client and server
2. **Ember-based callbacks** – Server-side processing: what happens when a command is received, how attribute memory is allocated

The configuration for each application (which endpoints, clusters, attributes are enabled) is stored in **`.zap` files**.

---

## Code Generation Inputs

### `.zap` Files

`.zap` files are large JSON files that configure the cluster selection for an application. They are edited using the **ZAP GUI tool**.

Each application has its own `.zap` file, typically in `examples/<app-name>/<app-name>-common/`.

**ZAP settings files** that define cluster metadata:
- `src/app/zap-templates/zcl/zcl.json` – default
- `src/app/zap-templates/zcl/zcl-with-test-extensions.json` – for `all-clusters-app`

### Editing `.zap` Files with the ZAP UI

```bash
# Make sure zap is in $PATH (or set ZAP_INSTALL_PATH / ZAP_DEVELOPMENT_PATH)
./scripts/tools/zap/run_zaptool.sh examples/lighting-app/lighting-common/lighting-app.zap
```

This launches the ZAP GUI with proper `--gen` and `--zcl` arguments pre-configured.

### ZAP Environment Variables (in order of precedence)

| Variable | Usage |
|----------|-------|
| `$ZAP_DEVELOPMENT_PATH` | ZAP source checkout (for ZAP development) |
| `$ZAP_INSTALL_PATH` | Path to unpacked ZAP release zip |
| *(none)* | Assumes `zap`/`zap-cli` is in `$PATH` (standard after bootstrap) |

---

## `.matter` IDL Files

`.matter` files are **human-readable** equivalents of `.zap` files. They look like a language IDL (similar to protobuf or AIDL):

- Co-located with `.zap` files in application directories
- Generated from `.zap` files during codegen
- Matter-specific (no Zigbee compatibility concerns)
- Used by Python-based codegen tools for faster, more flexible generation

### Example `.matter` file snippet

```
// Endpoint 0: Root Node
endpoint 0 {
  device type rootNode = 22;

  server cluster BasicInformation {
    attribute access(read: unsetPrivilege) dataModelRevision = 0;
    attribute vendorName = 1;
    attribute vendorID = 2;
    attribute productName = 3;
    ...
  }
}
```

### Matter IDL Format Details

The format is documented in [`scripts/py_matter_idl/matter/idl/README.md`](https://github.com/project-chip/connectedhomeip/blob/master/scripts/py_matter_idl/matter/idl/README.md).

---

## Code Generation Outputs

Generated code is organized into three categories:

| Category | Description | Location |
|----------|-------------|----------|
| **Application-specific** | Cluster callbacks, RAM allocation, attribute storage | Generated alongside `.zap` files in `examples/` |
| **Automated tests** | Test definition data for `chip-tool` and `darwin-framework-tool` | `examples/${TOOL}/templates/tests/` |
| **Controller clusters** | All-clusters selection for client-side access | `src/controller/data_model/` |

---

## Running Code Generation

### Regenerate All Code (Full)

```bash
./scripts/tools/zap_regen_all.py
```

This can take several minutes. To speed up test-only regeneration:

```bash
./scripts/tools/zap_regen_all.py --type tests
./scripts/tools/zap_regen_all.py --type tests --tests chip-tool
```

### Regenerate a Single Application

```bash
# Generates the .matter file alongside the .zap file
./scripts/tools/zap/generate.py examples/bridge-app/bridge-common/bridge-app.zap
```

### Using `codegen.py` (Python-based, Faster)

```bash
# Generate code from a .matter file
./scripts/codegen.py --input examples/lighting-app/lighting-common/lighting-app.matter \
                     --output-dir out/gen/lighting-app \
                     --template src/app/zap-templates/app-templates.json
```

---

## Pre-Generation for External Applications

For applications outside the SDK that use custom `.zap` files, pre-generation produces checked-in generated code.

### Step 1: Ensure You Have a `.matter` File

```bash
./scripts/tools/zap/generate.py path/to/your-app.zap
# This creates path/to/your-app.matter alongside the .zap file
```

### Step 2: Run Pre-Generation

```bash
./scripts/tools/zap/generate.py \
    --no-run-bootstrap \
    path/to/your-app.zap
```

### Using Pre-Generated Code

Reference the pre-generated directory in your `BUILD.gn`:

```gn
chip_data_model("my_app") {
    zap_file = "my-app.zap"
    zap_pregenerated_dir = "zap-generated"
}
```

---

## Matter IDL Tooling

### `matter-data-model-xml-parser`

Parses CSA XML data definitions and outputs `.matter` format:

```bash
# Parse a single cluster XML to .matter format
matter-data-model-xml-parser data_model/clusters/BooleanState.xml

# Output to a file
matter-data-model-xml-parser -o out/BooleanState.matter data_model/clusters/BooleanState.xml
```

### Comparing SDK Against Specification

```bash
matter-data-model-xml-parser \
     -o out/spec.matter \
     --compare-output out/sdk.matter \
     --compare src/controller/data_model/controller-clusters.matter \
     data_model/clusters/DoorLock.xml \
  && diff out/{spec,sdk}.matter
```

> **Note:** The scraper is still in development; manually validate the diff for tool errors or Zigbee-only markers.

### `matter-idl-lint`

Validates `.matter` files for device conformance (mandatory attributes, cluster requirements):

```bash
matter-idl-lint examples/window-app/common/window-app.matter
```

Rules are expressed in `.matterlint` files, loaded from Silabs XML files and hard-coded requirements.

---

## Python-based Codegen vs ZAP

| Feature | ZAP | Python (`codegen.py`) |
|---------|-----|----------------------|
| Third-party deps | Heavy (npm packages) | Minimal |
| Speed | Slow (minutes) | Fast |
| Flexibility | Limited | High (multi-file per cluster) |
| Template language | Handlebars | Python |
| Input format | `.zap` (JSON) | `.matter` (IDL) |
| Determinism | Occasionally non-deterministic | Deterministic |
| Complexity | Higher | Lower, typed, unit-tested |

Both methods are currently supported. The long-term goal is a single code generation method combining the benefits of both.

---

## Workflow Summary

```
Matter Spec (asciidoc)
        ↓ Alchemy tool
    XML Definitions (data_model/clusters/*.xml)
        ↓ ZAP UI editing
    .zap files (cluster selection per app)
        ↓ generate.py / zap_regen_all.py
    .matter files (human-readable IDL)
        ↓ codegen.py / ZAP code generation
    Generated C++ code (endpoint_config.h, callbacks, etc.)
```

---

## References

- [Code Generation Guide](https://project-chip.github.io/connectedhomeip-doc/zap_and_codegen/code_generation.html)
- [Matter IDL Guide](https://project-chip.github.io/connectedhomeip-doc/guides/matter_idl_tooling.html)
- [ZAP Repository](https://github.com/project-chip/zap)
- [Alchemy (spec parser)](https://github.com/project-chip/alchemy)
- [py_matter_idl README](https://github.com/project-chip/connectedhomeip/blob/master/scripts/py_matter_idl/matter/idl/README.md)
