# About

Modified `bzip2` recipe

* supports "openssl/OpenSSL_1_1_1-stable@conan/stable"
* supports sanitizers, see [https://github.com/google/sanitizers/wiki/MemorySanitizerLibcxxHowTo#instrumented-gtest](https://github.com/google/sanitizers/wiki/MemorySanitizerLibcxxHowTo#instrumented-gtest)
* uses `llvm_tools` conan package in builds with `LLVM_USE_SANITIZER`, see https://github.com/google/sanitizers/wiki/MemorySanitizerLibcxxHowTo#instrumented-gtest
* uses `llvm_tools` conan package in builds with libc++ (will be instrumented if `LLVM_USE_SANITIZER` enabled)
* etc.

NOTE: use `-s llvm_tools:build_type=Release` during `conan install`

# About

## Docker build

```bash
export MY_IP=$(ip route get 8.8.8.8 | sed -n '/src/{s/.*src *\([^ ]*\).*/\1/p;q}')
sudo -E docker build \
    --build-arg PKG_NAME=bzip2/1.0.8 \
    --build-arg PKG_CHANNEL=dev/stable \
    --build-arg PKG_UPLOAD_NAME=bzip2/1.0.8@dev/stable \
    --build-arg CONAN_EXTRA_REPOS="conan-local http://$MY_IP:8081/artifactory/api/conan/conan False" \
    --build-arg CONAN_EXTRA_REPOS_USER="user -p password1 -r conan-local admin" \
    --build-arg CONAN_INSTALL="conan install --profile clang --build missing" \
    --build-arg CONAN_CREATE="conan create --profile clang --build missing" \
    --build-arg CONAN_UPLOAD="conan upload --all -r=conan-local -c --retry 3 --retry-wait 10 --force" \
    --build-arg BUILD_TYPE=Debug \
    -f conan_bzip2.Dockerfile --tag conan_bzip2 . --no-cache
```

## Local build

```bash
conan remote add conan-center https://api.bintray.com/conan/conan/conan-center False

export PKG_NAME=bzip2/1.0.8@dev/stable

(CONAN_REVISIONS_ENABLED=1 \
    conan remove --force $PKG_NAME || true)

CONAN_REVISIONS_ENABLED=1 \
    CONAN_VERBOSE_TRACEBACK=1 \
    CONAN_PRINT_RUN_COMMANDS=1 \
    CONAN_LOGGING_LEVEL=10 \
    GIT_SSL_NO_VERIFY=true \
    conan create . \
      dev/stable \
      -s build_type=Release \
      --profile clang \
      --build missing \
      --build cascade

CONAN_REVISIONS_ENABLED=1 \
    CONAN_VERBOSE_TRACEBACK=1 \
    CONAN_PRINT_RUN_COMMANDS=1 \
    CONAN_LOGGING_LEVEL=10 \
    conan upload $PKG_NAME \
      --all -r=conan-local \
      -c --retry 3 \
      --retry-wait 10 \
      --force
```


## Build with sanitizers support

Use `-o llvm_tools:enable_tsan=True` and `-e *:compile_with_llvm_tools=True` like so:

```bash
export CC=$(find ~/.conan/data/llvm_tools/master/conan/stable/package/ -path "*bin/clang" | head -n 1)

export CXX=$(find ~/.conan/data/llvm_tools/master/conan/stable/package/ -path "*bin/clang++" | head -n 1)

export TSAN_OPTIONS="handle_segv=0:disable_coredump=0:abort_on_error=1:report_thread_leaks=0"

# make sure that env. var. TSAN_SYMBOLIZER_PATH points to llvm-symbolizer
# conan package llvm_tools provides llvm-symbolizer
# and prints its path during cmake configure step
# echo $TSAN_SYMBOLIZER_PATH
export TSAN_SYMBOLIZER_PATH=$(find ~/.conan/data/llvm_tools/master/conan/stable/package/ -path "*bin/llvm-symbolizer" | head -n 1)

# NOTE: NO `--profile` argument cause we use `CXX` env. var
CONAN_REVISIONS_ENABLED=1 \
    CONAN_VERBOSE_TRACEBACK=1 \
    CONAN_PRINT_RUN_COMMANDS=1 \
    CONAN_LOGGING_LEVEL=10 \
    GIT_SSL_NO_VERIFY=true \
    conan create . \
      dev/stable \
      -s build_type=Debug \
      -s compiler=clang \
      -s compiler.version=10 \
      -s compiler.libcxx=libc++ \
      -s llvm_tools:compiler=clang \
      -s llvm_tools:compiler.version=6.0 \
      -s llvm_tools:compiler.libcxx=libstdc++11 \
      -s llvm_tools:build_type=Release \
      -o llvm_tools:enable_tsan=True \
      -o llvm_tools:include_what_you_use=False \
      -e bzip2:compile_with_llvm_tools=True \
      -e bzip2:enable_llvm_tools=True \
      -o bzip2:enable_tsan=True
```

## How to diagnose errors in conanfile (CONAN_PRINT_RUN_COMMANDS)

```bash
# NOTE: about `--keep-source` see https://bincrafters.github.io/2018/02/27/Updated-Conan-Package-Flow-1.1/
CONAN_REVISIONS_ENABLED=1 CONAN_VERBOSE_TRACEBACK=1 CONAN_PRINT_RUN_COMMANDS=1 CONAN_LOGGING_LEVEL=10 conan create . conan/stable -s build_type=Debug --profile clang --build missing --build cascade --keep-source
```
