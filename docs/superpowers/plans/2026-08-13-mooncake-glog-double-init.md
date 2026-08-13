# Mooncake glog Double Initialization Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prevent Mooncake TransferEngine from aborting when `MC_LOG_DIR` is set and the embedding process has already initialized glog.

**Architecture:** Preserve the existing `loadGlobalConfig(GlobalConfig&)` interface and logging configuration flow. Add a process-state guard immediately around Mooncake's glog initialization call, then cover the exact host-first initialization order in a dedicated CTest executable so its global logging state is isolated from other tests.

**Tech Stack:** C++17, glog, GoogleTest, CMake/CTest, Mooncake TransferEngine, Python wheel integration on bt-6.200.22.140.

## Global Constraints

- Keep the change outside request, inference, and transfer hot paths.
- Do not change the TransferEngine ABI, library type, wheel layout, or Python extension load order.
- Preserve all existing `MC_LOG_DIR` validation and glog flag behavior.
- Do not modify Elastic EP control-plane, data-plane, or fault-tolerance state.
- Use `bt-6.200.22.140` and run all remote Git operations inside the existing `sgl0514-dev-wjl` container through the structured helper.
- Preserve the remote repository's existing `CMakeLists.txt`, `scripts/build_wheel.sh`, and `.epoch_probe_backup/` working-tree changes.

---

### Task 1: Prove the failure with an isolated regression

**Files:**
- Create: `mooncake-transfer-engine/tests/glog_preinitialized_test.cpp`
- Modify: `mooncake-transfer-engine/tests/CMakeLists.txt:307-310`

**Interfaces:**
- Consumes: `void mooncake::loadGlobalConfig(mooncake::GlobalConfig&)`, `google::InitGoogleLogging(const char*)`, and `google::IsGoogleLoggingInitialized()`.
- Produces: CTest target `glog_preinitialized_test`; no production interface changes.

- [ ] **Step 1: Add a dedicated failing regression test**

Create `mooncake-transfer-engine/tests/glog_preinitialized_test.cpp`:

```cpp
// Copyright 2026 KVCache.AI
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

#include <glog/logging.h>
#include <gtest/gtest.h>

#include <cstdlib>

#include "config.h"

namespace mooncake {
namespace {

class GlogPreinitializedTest : public ::testing::Test {
   protected:
    void TearDown() override {
        ::unsetenv("MC_LOG_DIR");
        if (google::IsGoogleLoggingInitialized()) {
            google::ShutdownGoogleLogging();
        }
    }
};

TEST_F(GlogPreinitializedTest, LoadGlobalConfigReusesHostLogging) {
    google::InitGoogleLogging("glog-preinitialized-test");
    ASSERT_TRUE(google::IsGoogleLoggingInitialized());
    ASSERT_EQ(::setenv("MC_LOG_DIR", ".", 1), 0);

    GlobalConfig config;
    loadGlobalConfig(config);

    EXPECT_TRUE(google::IsGoogleLoggingInitialized());
    EXPECT_EQ(FLAGS_log_dir, ".");
}

}  // namespace
}  // namespace mooncake
```

Register it in `mooncake-transfer-engine/tests/CMakeLists.txt`:

```cmake
add_executable(glog_preinitialized_test
               ${WORKSPACE}/glog_preinitialized_test.cpp)
target_link_libraries(glog_preinitialized_test PUBLIC transfer_engine gtest
                                                      gtest_main glog::glog)
add_test(NAME glog_preinitialized_test COMMAND glog_preinitialized_test)
```

- [ ] **Step 2: Format, inspect, and commit the regression test**

Run:

```bash
git add mooncake-transfer-engine/tests/CMakeLists.txt \
  mooncake-transfer-engine/tests/glog_preinitialized_test.cpp
./scripts/code_format.sh --staged
git diff --cached --check
git diff --cached
git commit -m "[TransferEngine] Reproduce duplicate glog initialization"
```

Expected: the commit contains only the new test source and CMake registration.

- [ ] **Step 3: Push and pull the test commit through the approved Git routes**

Push local branch `wjl/github-main-20260813` to Coding JD. In
`sgl0514-dev-wjl`, verify the three known pre-existing remote changes do not
overlap the test files, then fast-forward
`/ufs/wjl/workspace/JD-AI-Infra-Mooncake-ElasticEP/Mooncake` with:

```bash
git -C /ufs/wjl/workspace/JD-AI-Infra-Mooncake-ElasticEP/Mooncake pull --ff-only origin wjl/github-main-20260813
```

Expected: the test commit is present and the existing root `CMakeLists.txt`,
`scripts/build_wheel.sh`, and `.epoch_probe_backup/` changes remain untouched.

- [ ] **Step 4: Configure the persistent regression build and prove RED**

Inside the existing `sgl0514-dev-wjl` container, run:

```bash
cmake -S . -B build-glog-regression \
  -DBUILD_UNIT_TESTS=ON \
  -DBUILD_EXAMPLES=OFF \
  -DBUILD_BENCHMARK=OFF \
  -DWITH_STORE=OFF \
  -DWITH_STORE_RUST=OFF \
  -DWITH_P2P_STORE=OFF \
  -DWITH_EP=OFF \
  -DUSE_CUDA=OFF \
  -DUSE_CXL=OFF \
  -DUSE_HTTP=OFF \
  -DUSE_ETCD=OFF \
  -DSTORE_USE_ETCD=OFF \
  -DUSE_MNNVL=OFF \
  -DUSE_INTRA_NVLINK=OFF
cmake --build build-glog-regression --target glog_preinitialized_test -j2
ctest --test-dir build-glog-regression --output-on-failure \
  -R '^glog_preinitialized_test$'
```

Expected: configure and build succeed, then the test process aborts with
`You called InitGoogleLogging() twice!`, proving the test exercises the
production failure.

---

### Task 2: Add the minimal glog initialization guard

**Files:**
- Modify: `mooncake-transfer-engine/src/config.cpp:444-447`

**Interfaces:**
- Consumes: `google::IsGoogleLoggingInitialized()` and the regression target from Task 1.
- Produces: guarded glog initialization with unchanged `loadGlobalConfig(GlobalConfig&)` signature and configuration semantics.

- [ ] **Step 1: Add the minimal production guard**

Change only the initialization call in `mooncake-transfer-engine/src/config.cpp`:

```cpp
const char* log_dir_path = std::getenv("MC_LOG_DIR");
if (log_dir_path) {
    if (!google::IsGoogleLoggingInitialized()) {
        google::InitGoogleLogging("mooncake-transfer-engine");
    }
```

Leave the existing directory checks and flag assignments unchanged.

- [ ] **Step 2: Format, inspect, and commit the production change**

Run:

```bash
git add mooncake-transfer-engine/src/config.cpp
./scripts/code_format.sh --staged
git diff --cached --check
git diff --cached
git commit -m "[TransferEngine] Avoid duplicate glog initialization"
```

Expected: the commit changes only the existing unconditional initialization into a guarded initialization.

- [ ] **Step 3: Push, pull, and prove GREEN**

Push the branch, fast-forward the remote container repository with the same
approved Git routes as Task 1, then run:

```bash
cmake --build build-glog-regression --target glog_preinitialized_test \
  config_test -j2
ctest --test-dir build-glog-regression --output-on-failure \
  -R '^(glog_preinitialized_test|config_test)$'
```

Expected: both tests pass; `glog_preinitialized_test` confirms host-owned glog remains initialized and `FLAGS_log_dir` remains `.`.

---

### Task 3: Run production-oriented regression checks in the bt container

**Files:**
- Verify: `mooncake-transfer-engine/src/config.cpp`
- Verify: `mooncake-transfer-engine/tests/glog_preinitialized_test.cpp`
- Preserve: remote `CMakeLists.txt`, `scripts/build_wheel.sh`, `.epoch_probe_backup/`

**Interfaces:**
- Consumes: committed branch `wjl/github-main-20260813`, configured `build-glog-regression`, and the structured WJL remote helper.
- Produces: Release build evidence, focused CTest evidence, repository provenance, and a documented end-to-end boundary.

- [ ] **Step 1: Build the production TransferEngine and integration extension**

Configure a Release build in the same persistent directory and compile the
production targets without starting or replacing a container:

```bash
cmake -S . -B build-glog-regression -DCMAKE_BUILD_TYPE=Release
cmake --build build-glog-regression --target transfer_engine engine -j2
```

Expected: both Release targets compile successfully against the same source as the tests.

- [ ] **Step 2: Re-run the focused tests after the Release target build**

```bash
ctest --test-dir build-glog-regression --output-on-failure \
  -R '^(glog_preinitialized_test|config_test)$'
```

Expected: both tests still pass after the production targets are rebuilt.

- [ ] **Step 3: Verify provenance and preserved remote state**

Run inside `sgl0514-dev-wjl`:

```bash
git -C /ufs/wjl/workspace/JD-AI-Infra-Mooncake-ElasticEP/Mooncake rev-parse HEAD
git -C /ufs/wjl/workspace/JD-AI-Infra-Mooncake-ElasticEP/Mooncake status --short
```

Expected: HEAD equals Coding JD's branch tip; only the pre-existing root
`CMakeLists.txt`, `scripts/build_wheel.sh`, and `.epoch_probe_backup/` appear.

- [ ] **Step 4: Record evidence and remaining end-to-end boundary**

Report every command's exit code, RED and GREEN CTest results, Release target
build results, local commits, remote HEAD, and preserved dirty paths. Do not run
the benchmark harness, rebuild its wheel through a new Docker container, or
restart the 1P1D service without separate cluster-mutation authorization.
