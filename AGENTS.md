# AGENTS.md - AI Agent Guidelines for OneWifi

This document provides guidelines for AI coding agents working in the OneWifi repository.

## Project Overview

OneWifi is a unified Wi-Fi management architecture for RDK (Reference Design Kit) devices.
It manages Wi-Fi parameters, statistics, telemetry, steering, and optimization for both
Gateways and Extenders.

- **Language**: C (primary), C++ for tests and some utilities
- **Framework**: RDK/CCSP ecosystem
- **License**: Apache License 2.0

## Build Commands

### Autotools Build (Primary)
```bash
./autogen.sh
./configure [options]
make
```

### Linux Platform Build (Raspberry Pi / Banana Pi)
```bash
make -f build/linux/rpi/makefile setup
make -f build/linux/rpi/makefile all
```

### Configure Options
- `--enable-gtestapp` - Enable Google Test support
- `--enable-libwebconfig` - Enable libwebconfig
- `--enable-easymesh` - Enable EasyMesh
- `--enable-sm-app` - Enable Statistics Manager app
- `--enable-em-app` - Enable EasyMesh app
- `--with-ccsp-arch={arm,atom,pc,mips}` - Set CPU architecture

## Testing

### Test Framework
Google Test (gtest) + Google Mock (gmock)

### Building Tests
```bash
./configure --enable-gtestapp
make
```

### Running All Tests
```bash
./OneWifi_gtest.bin
```

### Running a Single Test
```bash
./OneWifi_gtest.bin --gtest_filter=TestSuiteName.TestName
./OneWifi_gtest.bin --gtest_filter=TestSuiteName.*      # All tests in suite
./OneWifi_gtest.bin --gtest_filter=*Pattern*            # Pattern matching
```

### Test Output
XML reports: `/tmp/Gtest_Report/OneWifi_gtest_report.xml`

## Code Style

Use `clang-format` with the `.clang-format` file in the repository root.

### Formatting Basics
- **Indentation**: 4 spaces (no tabs)
- **Line length**: 100 characters max
- **Braces**: K&R style for control statements, opening brace on new line for functions
- **Pointers**: Asterisk aligned to variable name: `char *ptr`

### Naming Conventions
- **Variables/Functions**: `lowercase_with_underscores`
- **Macros/Enums**: `UPPERCASE_WITH_UNDERSCORES`
- **Typedefs**: Suffix with `_t` (e.g., `my_struct_t`)
- **Function pointer typedefs**: Suffix with `_fn` (e.g., `callback_fn`)

### Function Style
```c
int my_func(char *param1, int param2)
{
    int result = 0;

    if (param1 == NULL) {
        return -1;
    }

    result = do_something(param1, param2);

    return result;
}
```

### Control Statements
- Always use braces, even for single-line bodies
- `else` on same line as closing brace: `} else {`
- Space after keywords: `if (`, `for (`, `while (`

### Variables
- Declare at beginning of block, before first executable statement
- Add one empty line after variable declarations
- Do not align variable assignments

### Switch Statements
- No indent for `case` labels
- Always include `default`
- Put `break` inside case blocks with local variables

## Error Handling

### Return Values
- Success/fail functions: `0` for success, `-1` for failure
- Pointer functions: `NULL` on failure
- Boolean functions: `true`/`false`

### Goto Pattern for Cleanup
```c
int my_func(void)
{
    void *p1 = NULL, *p2 = NULL;
    int ret = -1;

    p1 = malloc(256);
    if (p1 == NULL) {
        return -1;
    }

    p2 = malloc(512);
    if (p2 == NULL) {
        goto exit;
    }

    // ... work ...
    ret = 0;

exit:
    free(p2);
    free(p1);
    return ret;
}
```

### Early Exit Strategy
Check preconditions and return early to avoid deep nesting.

## Logging

Use the logging macros from `wifi_util.h`:
```c
wifi_util_error_print(WIFI_CTRL, "%s:%d error message\n", __func__, __LINE__);
wifi_util_info_print(WIFI_CTRL, "%s:%d info message\n", __func__, __LINE__);
wifi_util_dbg_print(WIFI_CTRL, "%s:%d debug message\n", __func__, __LINE__);
```

Error messages should contain "failed" or "error" for easy log scanning.

## Header Files

### Include Guards and C++ Check
```c
#ifndef MODULE_NAME_H
#define MODULE_NAME_H

#ifdef __cplusplus
extern "C" {
#endif

/* declarations */

#ifdef __cplusplus
}
#endif

#endif /* MODULE_NAME_H */
```

### Include Order
1. Component header (e.g., `foo.h` in `foo.c`)
2. Application headers
3. System headers

Use `<>` for standard library, `""` for project headers.

## Safety Guidelines

- Use safe string functions: `snprintf()`, `strncpy()` instead of `sprintf()`, `strcpy()`
- Use `static` for file-scoped functions and variables
- Initialize pointers to NULL
- Always check return values and NULL pointers
- Functions should not exceed 80 lines

## License Header

Every source file must include the Apache 2.0 license header:
```c
/************************************************************************************
  If not stated otherwise in this file or this component's LICENSE file the
  following copyright and licenses apply:

  Copyright 2018 RDK Management

  Licensed under the Apache License, Version 2.0 (the "License");
  ...
 **************************************************************************/
```

## Project Structure

```
source/
├── core/       # Core wifi management (wifi_ctrl.c, wifi_mgr.c)
├── apps/       # Application modules (analytics, blaster, csi, em)
├── db/         # Database layer
├── dml/        # Data Model Layer (TR-181)
├── platform/   # Platform-specific code (linux, rdkb, extender)
├── stats/      # Statistics collection
├── test/       # Unit tests (GTest)
├── utils/      # Utilities (scheduler, collection)
└── webconfig/  # Web configuration encoding/decoding
lib/            # Internal libraries (common, ds, log, ovsdb, json_util)
include/        # Public headers
build/          # Build configurations (linux, openwrt)
```

## Contributing

1. Fork the repository and send pull requests
2. Sign the RDK Contributor License Agreement (CLA)
3. Follow code style guidelines
4. Run `clang-format` before committing
5. Include proper license headers in all files
