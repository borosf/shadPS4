<!--
SPDX-FileCopyrightText: 2024 shadPS4 Emulator Project
SPDX-License-Identifier: GPL-2.0-or-later
-->

# Security Fixes Required for shadPS4

**Priority**: CRITICAL  
**Date**: January 2026

This document outlines specific security vulnerabilities that require immediate attention and provides concrete fix recommendations.

---

## 🔴 CRITICAL: Buffer Overflow Vulnerabilities

### Fix #1: Network Interface Name Copy
**File**: `src/core/libraries/network/net_util.cpp:103`

**Current (VULNERABLE)**:
```cpp
strcpy(ifr.ifr_name, it->ifr_name);
```

**Fixed**:
```cpp
strncpy(ifr.ifr_name, it->ifr_name, IFNAMSIZ - 1);
ifr.ifr_name[IFNAMSIZ - 1] = '\0';  // Ensure null termination
```

---

### Fix #2: IPv6 Address String Copy
**File**: `src/core/libraries/network/net.cpp:1192`

**Current (VULNERABLE)**:
```cpp
if ((u64)(tp - tmp) > size)
    return nullptr;
strcpy(dst, tmp);  // Still unsafe!
```

**Fixed**:
```cpp
if ((u64)(tp - tmp) > size)
    return nullptr;
strncpy(dst, tmp, size - 1);
dst[size - 1] = '\0';
```

---

### Fix #3: DNS Server Placeholders
**File**: `src/core/libraries/network/netctl.cpp:187-191`

**Current (VULNERABLE)**:
```cpp
strcpy(info->ip_address, "127.0.0.1");
strcpy(info->primary_dns, "1.1.1.1");
strcpy(info->secondary_dns, "1.1.1.1");
```

**Fixed**:
```cpp
// Assuming SCE_NET_CTL_IPV4_ADDR_STR_LEN is 16
strncpy(info->ip_address, "127.0.0.1", SCE_NET_CTL_IPV4_ADDR_STR_LEN - 1);
info->ip_address[SCE_NET_CTL_IPV4_ADDR_STR_LEN - 1] = '\0';

strncpy(info->primary_dns, "1.1.1.1", SCE_NET_CTL_IPV4_ADDR_STR_LEN - 1);
info->primary_dns[SCE_NET_CTL_IPV4_ADDR_STR_LEN - 1] = '\0';

strncpy(info->secondary_dns, "1.1.1.1", SCE_NET_CTL_IPV4_ADDR_STR_LEN - 1);
info->secondary_dns[SCE_NET_CTL_IPV4_ADDR_STR_LEN - 1] = '\0';
```

**Or Better (Modern C++)**:
```cpp
// If the struct can be modified to use std::array or if you can use a helper
auto safe_copy = [](char* dst, size_t dst_size, const char* src) {
    strncpy(dst, src, dst_size - 1);
    dst[dst_size - 1] = '\0';
};

safe_copy(info->ip_address, sizeof(info->ip_address), "127.0.0.1");
safe_copy(info->primary_dns, sizeof(info->primary_dns), "1.1.1.1");
safe_copy(info->secondary_dns, sizeof(info->secondary_dns), "1.1.1.1");
```

---

### Fix #4: Internal libc Wrapper Safety
**File**: `src/core/libraries/libc_internal/libc_internal_str.cpp:16, 25, 50`

**Current (VULNERABLE on non-Windows)**:
```cpp
s32 internal_strcpy_s(char* dest, size_t dest_size, const char* src) {
#if _WIN32
    return strcpy_s(dest, dest_size, src);
#else
    std::strcpy(dest, src);  // Ignores dest_size!
    return 0;
#endif
}
```

**Fixed**:
```cpp
s32 internal_strcpy_s(char* dest, size_t dest_size, const char* src) {
#if _WIN32
    return strcpy_s(dest, dest_size, src);
#else
    if (!dest || !src || dest_size == 0) {
        return EINVAL;
    }
    
    size_t src_len = strlen(src);
    if (src_len >= dest_size) {
        // Buffer too small
        if (dest_size > 0) {
            dest[0] = '\0';
        }
        return ERANGE;
    }
    
    strncpy(dest, src, dest_size - 1);
    dest[dest_size - 1] = '\0';
    return 0;
#endif
}
```

**Or Use Existing Safe Functions**:
```cpp
#if !defined(_WIN32) && defined(__STDC_LIB_EXT1__)
    // Use C11 strcpy_s if available
    return strcpy_s(dest, dest_size, src);
#elif !defined(_WIN32) && defined(__BSD__)
    // Use BSD strlcpy if available
    if (strlcpy(dest, src, dest_size) >= dest_size) {
        return ERANGE;
    }
    return 0;
#else
    // Fallback implementation (as above)
#endif
```

---

## 🔴 CRITICAL: Memory Allocation Bug

### Fix #5: AIO Module malloc/memset Bug
**File**: `src/core/libraries/kernel/aio.cpp:317-318`

**Current (CRITICAL BUG)**:
```cpp
id_state = (int*)malloc(sizeof(int) * MAX_QUEUE);
memset(id_state, 0, sizeof(sizeof(int) * MAX_QUEUE));  // BUG!
```

**Problems**:
1. No null check after `malloc()`
2. `memset()` has double `sizeof()` → only clears 8 bytes instead of full array
3. Used throughout without null checks

**Fixed**:
```cpp
id_state = (int*)malloc(sizeof(int) * MAX_QUEUE);
if (!id_state) {
    LOG_ERROR(Lib_Kernel, "Failed to allocate AIO id_state array");
    return SCE_KERNEL_ERROR_ENOMEM;
}
memset(id_state, 0, sizeof(int) * MAX_QUEUE);  // Fixed: removed double sizeof
```

**Or Modern C++**:
```cpp
id_state = new (std::nothrow) int[MAX_QUEUE]();  // Value-initialized to 0
if (!id_state) {
    LOG_ERROR(Lib_Kernel, "Failed to allocate AIO id_state array");
    return SCE_KERNEL_ERROR_ENOMEM;
}
```

**Or Even Better (RAII)**:
```cpp
// At file scope, replace static pointer with static vector
static std::vector<int> id_state(MAX_QUEUE, 0);

// Now initialization is automatic and safe
// Access still works: id_state[index]
```

---

## 🔴 CRITICAL: Memory Leaks

### Fix #6: ImGui Initialization Leaks
**File**: `src/imgui/renderer/imgui_core.cpp:57-64`

**Current (MEMORY LEAK)**:
```cpp
char* config_file_buf = new char[path.size() + 1]();
std::memcpy(config_file_buf, path.c_str(), path.size());
io.IniFilename = config_file_buf;  // Never freed!

char* log_file_buf = new char[path.size() + 1]();
std::memcpy(log_file_buf, path.c_str(), path.size());
io.LogFilename = log_file_buf;  // Never freed!
```

**Option 1: Class Member Storage**
```cpp
// In imgui_core.h, add private members:
private:
    std::string m_config_file_path;
    std::string m_log_file_path;

// In imgui_core.cpp:
m_config_file_path = user_dir / "imgui.ini";
m_log_file_path = user_dir / "imgui_log.txt";

io.IniFilename = m_config_file_path.c_str();
io.LogFilename = m_log_file_path.c_str();
```

**Option 2: Static Storage**
```cpp
// If paths are known at compile time or don't change
static const std::string config_path = (user_dir / "imgui.ini").string();
static const std::string log_path = (user_dir / "imgui_log.txt").string();

io.IniFilename = config_path.c_str();
io.LogFilename = log_path.c_str();
```

**Option 3: Verify ImGui Ownership**
```cpp
// Check if ImGui actually takes ownership and frees these
// If it does, document it with a comment:

// ImGui takes ownership and will free these on shutdown
char* config_file_buf = new char[path.size() + 1]();
std::memcpy(config_file_buf, path.c_str(), path.size());
io.IniFilename = config_file_buf;
```

---

## 🟠 HIGH: Error Handling

### Fix #7: Socket Operation Error Checking
**File**: `src/core/libraries/network/sys_net.cpp`

**Current (MISSING ERROR CHECKS)**:
```cpp
listener = socket(AF_INET, SOCK_STREAM, 0);
if (bind(listener, (sockaddr*)&addr, ...) == SOCKET_ERROR) {
    // handle error
}
listen(listener, 1);
sock1 = socket(AF_INET, SOCK_STREAM, 0);
sock2 = accept(listener, nullptr, nullptr);
```

**Fixed**:
```cpp
listener = socket(AF_INET, SOCK_STREAM, 0);
if (listener == INVALID_SOCKET) {
    LOG_ERROR(Lib_Net, "Failed to create listener socket: {}", GetLastSocketError());
    return SCE_NET_ERROR_SOCKET_CREATION_FAILED;
}

if (bind(listener, (sockaddr*)&addr, sizeof(addr)) == SOCKET_ERROR) {
    LOG_ERROR(Lib_Net, "Failed to bind listener socket: {}", GetLastSocketError());
    closesocket(listener);
    return SCE_NET_ERROR_BIND_FAILED;
}

if (listen(listener, 1) == SOCKET_ERROR) {
    LOG_ERROR(Lib_Net, "Failed to listen on socket: {}", GetLastSocketError());
    closesocket(listener);
    return SCE_NET_ERROR_LISTEN_FAILED;
}

sock1 = socket(AF_INET, SOCK_STREAM, 0);
if (sock1 == INVALID_SOCKET) {
    LOG_ERROR(Lib_Net, "Failed to create sock1: {}", GetLastSocketError());
    closesocket(listener);
    return SCE_NET_ERROR_SOCKET_CREATION_FAILED;
}

sock2 = accept(listener, nullptr, nullptr);
if (sock2 == INVALID_SOCKET) {
    LOG_ERROR(Lib_Net, "Failed to accept connection: {}", GetLastSocketError());
    closesocket(listener);
    closesocket(sock1);
    return SCE_NET_ERROR_ACCEPT_FAILED;
}
```

---

### Fix #8: pclose(nullptr) Logic Bug
**File**: `src/core/devtools/widget/common.h:120-124`

**Current (LOGIC BUG)**:
```cpp
const auto f = popen(cli, "r");
if (!f) {
    pclose(f);  // BUG: f is nullptr here!
    return {};
}
```

**Fixed**:
```cpp
const auto f = popen(cli, "r");
if (!f) {
    LOG_ERROR(DevTools, "Failed to execute command: {}", cli);
    return {};  // Just return, don't call pclose
}

// ... read output ...

pclose(f);  // Only close if it was successfully opened
return result;
```

---

## 🟠 HIGH: Thread Safety

### Fix #9: AIO Module Race Conditions
**File**: `src/core/libraries/kernel/aio.cpp`

**Current (DATA RACE)**:
```cpp
static s32* id_state;           // No synchronization
static s32 id_index;            // Incremented without locks
```

**Fixed**:
```cpp
static std::mutex aio_mutex;    // Add mutex
static s32* id_state;
static s32 id_index;

// In functions that access these globals:
s32 sceKernelAioSubmitReadCommands(...) {
    std::scoped_lock lock(aio_mutex);
    
    // Now safe to access id_state and id_index
    s32 id = id_index;
    id_index = (id_index + 1) % MAX_QUEUE;
    
    id_state[id] = ...;
    
    // ... rest of function ...
}
```

**Or Better (Thread-Local)**:
```cpp
// If each thread should have its own AIO context
static thread_local s32 id_state[MAX_QUEUE] = {};
static thread_local s32 id_index = 0;
```

---

## 🟡 MEDIUM: Build System Security

### Fix #10: Enable Sanitizers for Debug Builds
**File**: `CMakeLists.txt`

**Add**:
```cmake
# Enable sanitizers for Debug builds
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
        add_compile_options(-fsanitize=address,undefined)
        add_link_options(-fsanitize=address,undefined)
        message(STATUS "Sanitizers enabled: address, undefined")
    endif()
endif()

# Optional: Add Thread Sanitizer (separate builds, incompatible with ASan)
option(ENABLE_TSAN "Enable Thread Sanitizer" OFF)
if(ENABLE_TSAN)
    add_compile_options(-fsanitize=thread)
    add_link_options(-fsanitize=thread)
    message(STATUS "Thread Sanitizer enabled")
endif()
```

---

### Fix #11: Audit MSVC Security Warnings
**File**: `CMakeLists.txt`

**Current**:
```cmake
add_definitions(
    -D_CRT_SECURE_NO_WARNINGS
    -D_CRT_NONSTDC_NO_DEPRECATE
    -D_SCL_SECURE_NO_WARNINGS
)
```

**Recommended**: Remove these and fix warnings individually, or at minimum:
```cmake
# Only disable for external dependencies
target_compile_definitions(external_library PRIVATE
    _CRT_SECURE_NO_WARNINGS
)

# For main codebase, use secure functions
# Example: Replace sprintf with sprintf_s, strcpy with strcpy_s
```

---

## 🟢 LOW: Code Quality

### Fix #12: String Operation Optimization
**File**: `src/core/file_sys/fs.cpp:15-22`

**Current (INEFFICIENT)**:
```cpp
std::string RemoveTrailingSlashes(const std::string& path) {
    std::string path_sanitized = path;  // Unnecessary copy
    while (path_sanitized.ends_with("/")) {
        path_sanitized.pop_back();
    }
    return path_sanitized;
}
```

**Fixed (Option 1 - In-place)**:
```cpp
std::string RemoveTrailingSlashes(std::string path) {  // Take by value for RVO
    while (!path.empty() && path.back() == '/') {
        path.pop_back();
    }
    return path;
}
```

**Fixed (Option 2 - View-based)**:
```cpp
std::string_view RemoveTrailingSlashes(std::string_view path) {
    while (!path.empty() && path.back() == '/') {
        path.remove_suffix(1);
    }
    return path;
}
```

---

## Verification Steps

After applying these fixes:

1. **Build with Sanitizers**:
   ```bash
   cmake -B build -DCMAKE_BUILD_TYPE=Debug
   cmake --build build
   ./build/shadPS4 <test_game>
   ```

2. **Run Static Analysis**:
   ```bash
   clang-tidy src/**/*.cpp -- -std=c++23
   cppcheck --enable=all --inconclusive src/
   ```

3. **Test Network Code**:
   - Run games with network functionality
   - Monitor for sanitizer warnings
   - Check logs for socket errors

4. **Test AIO Operations**:
   - Run games that use async I/O
   - Test with Thread Sanitizer
   - Verify no race conditions

5. **Memory Leak Detection**:
   ```bash
   valgrind --leak-check=full ./build/shadPS4 <test_game>
   ```

---

## Timeline Recommendation

- **Week 1**: Fix all CRITICAL issues (#1-6)
- **Week 2**: Fix all HIGH issues (#7-9)
- **Week 3**: Build system improvements (#10-11)
- **Week 4**: Testing and verification

---

**End of Security Fixes Document**
