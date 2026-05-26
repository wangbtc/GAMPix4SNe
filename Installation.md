# Installation Guide: ROOT, Geant4, and edep-sim on Ubuntu

This guide covers building ROOT, Geant4, and edep-sim from source on Ubuntu 20.04 (Focal),
installing everything into a shared directory so multiple users on the same machine can access them.

---

## Table of Contents

- [Directory Structure](#directory-structure)
- [System Dependencies](#system-dependencies)
- [Install ROOT](#install-root)
- [Install Geant4](#install-geant4)
- [Install edep-sim](#install-edep-sim)
- [Shared Environment Setup](#shared-environment-setup)
- [Troubleshooting: Common Ubuntu Issues](#troubleshooting-common-ubuntu-issues)

---

## Directory Structure

All packages are installed under a shared path so any user on the machine can source the environment.

```bash
mkdir -p /media/uadm/data5/GAMPix4SNe/build/root
mkdir -p /media/uadm/data5/GAMPix4SNe/build/geant4
mkdir -p /media/uadm/data5/GAMPix4SNe/build/edep-sim
mkdir -p /media/uadm/data5/GAMPix4SNe/opt
```

Adjust the base path `/media/uadm/data5/GAMPix4SNe` to wherever your shared storage lives.

---

## System Dependencies

Install all required build dependencies before starting:

```bash
sudo apt update
sudo apt install -y \
  build-essential cmake git \
  python3-dev libssl-dev \
  libx11-dev libxext-dev libxft-dev libxpm-dev \
  libxmu-dev libxi-dev \
  libglu1-mesa-dev libgl1-mesa-dev \
  qtbase5-dev libqt5opengl5-dev \
  libxerces-c-dev libexpat1-dev \
  zlib1g-dev libfreetype6-dev libpcre3-dev \
  libtbb-dev libfftw3-dev libgsl-dev \
  doxygen
```

> **Note:** Ubuntu 20.04 ships with Python 3.8, which is too old for ROOT 6.36+.
> Install Python 3.9 from the deadsnakes PPA (see [Troubleshooting](#troubleshooting-common-ubuntu-issues)).

---

## Install ROOT

**Versions tested:** ROOT 6.36.10

### Download source

```bash
cd ~/source
wget https://root.cern/download/root_v6.36.10.source.tar.gz
tar xzf root_v6.36.10.source.tar.gz
```

### Configure and build

```bash
cd /media/uadm/data5/GAMPix4SNe/build/root

cmake \
  -DCMAKE_INSTALL_PREFIX=/media/uadm/data5/GAMPix4SNe/opt/root-v6.36.10 \
  -Dbuiltin_vdt=ON \
  -DPYTHON_EXECUTABLE=/usr/bin/python3.9 \
  -DPython3_EXECUTABLE=/usr/bin/python3.9 \
  ~/source/root-6.36.10

cmake --build . -- -j$(nproc)
make install
```

---

## Install Geant4

**Versions tested:** Geant4 10.7.4

### Download source

```bash
cd ~/source
wget https://gitlab.cern.ch/geant4/geant4/-/archive/v10.7.4/geant4-v10.7.4.tar.gz
tar xzf geant4-v10.7.4.tar.gz
```

### Configure and build

```bash
cd /media/uadm/data5/GAMPix4SNe/build/geant4

cmake \
  -DCMAKE_INSTALL_PREFIX=/media/uadm/data5/GAMPix4SNe/opt/geant4-v10.7.4 \
  -DCMAKE_POLICY_DEFAULT_CMP0135=NEW \
  -DGEANT4_INSTALL_DATA=ON \
  -DGEANT4_USE_GDML=ON \
  -DGEANT4_USE_QT=ON \
  -DGEANT4_USE_RAYTRACER_X11=ON \
  -DGEANT4_USE_OPENGL_X11=ON \
  -DGEANT4_BUILD_MULTITHREADED=ON \
  -DGEANT4_USE_SYSTEM_ZLIB=ON \
  ~/source/geant4-v10.7.4

make -j$(nproc)
make install
```

> **Note:** On Ubuntu, `libxerces-c-dev` is system-installed, so no manual `-DXercesC_*` flags are needed
> (unlike the macOS build instructions that are sometimes bundled with downstream packages).

---

## Install edep-sim

**Repository:** https://github.com/yuntsebaryon/edep-sim-origin

### Source the environment first

ROOT and Geant4 must be in the environment before configuring edep-sim:

```bash
source /media/uadm/data5/GAMPix4SNe/opt/root-v6.36.10/bin/thisroot.sh
source /media/uadm/data5/GAMPix4SNe/opt/geant4-v10.7.4/bin/geant4.sh
```

### Clone and set up

```bash
cd /media/uadm/data5/GAMPix4SNe
git clone https://github.com/yuntsebaryon/edep-sim-origin.git

cd edep-sim-origin
source setup.sh
echo $EDEP_TARGET   # e.g. edep-gcc-9.4.0-x86_64-linux-gnu
```

### Configure and build

```bash
cd /media/uadm/data5/GAMPix4SNe/build/edep-sim

cmake \
  -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
  -DCMAKE_INSTALL_PREFIX=/media/uadm/data5/GAMPix4SNe/edep-sim-origin/${EDEP_TARGET} \
  /media/uadm/data5/GAMPix4SNe/edep-sim-origin

make -j$(nproc)
make install
```

---

## Shared Environment Setup

Create a single `setup.sh` at the top level that any user can source:

```bash
cat > /media/uadm/data5/GAMPix4SNe/setup.sh << 'EOF'
source /media/uadm/data5/GAMPix4SNe/opt/root-v6.36.10/bin/thisroot.sh
source /media/uadm/data5/GAMPix4SNe/opt/geant4-v10.7.4/bin/geant4.sh

cd /media/uadm/data5/GAMPix4SNe/edep-sim-origin
source setup.sh
export CMAKE_PREFIX_PATH=/media/uadm/data5/GAMPix4SNe/edep-sim-origin/${EDEP_TARGET}:$CMAKE_PREFIX_PATH
cd - > /dev/null
EOF
```

Any user can activate the full environment with:

```bash
source /media/uadm/data5/GAMPix4SNe/setup.sh
```

---

## Troubleshooting: Common Ubuntu Issues

### 1. Stale or broken apt repository (CERN CernVM-FS)

**Symptom:**
```
E: The repository 'http://cvmrepo.s3.cern.ch/cvmrepo/apt focal-prod Release' no longer has a Release file.
```

**Cause:** The `cernvm.list` symlink (managed by the `cvmfs-release` package) points to a dead S3-hosted repo.

**Fix:**
```bash
sudo rm /etc/apt/sources.list.d/cernvm.list
sudo rm -f /etc/apt/sources.list.d/cernvm.list.save
sudo apt update
```

---

### 2. ROOT build fails: `PyObject_CallOneArg` not declared

**Symptom:**
```
error: 'PyObject_CallOneArg' was not declared in this scope
```

**Cause:** `PyObject_CallOneArg` was introduced in Python 3.9. Ubuntu 20.04 defaults to Python 3.8.

**Fix:** Install Python 3.9 from the deadsnakes PPA and point cmake to it:

```bash
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.9 python3.9-dev python3.9-distutils
```

Then reconfigure ROOT with:
```bash
-DPYTHON_EXECUTABLE=/usr/bin/python3.9 \
-DPython3_EXECUTABLE=/usr/bin/python3.9
```

---

### 3. Geant4 cmake fails: missing Xmu library

**Symptom:**
```
CMake Error: could not find X11 Xmu library and/or headers
```

**Fix:**
```bash
sudo apt install libxmu-dev libxi-dev
```

---

### 4. edep-sim cmake fails: `geant4_set_and_check_package_variable` macro error

**Symptom:**
```
CMake Error at Geant4PackageCache.cmake:19:
  geant4_set_and_check_package_variable Macro invoked with incorrect arguments
```

**Cause:** Geant4 10.7.4's generated `Geant4PackageCache.cmake` calls the macro with an empty
`EXPAT_LIBRARY` entry that has no type argument — cmake 3.27+ rejects this.

**Fix:** Patch the installed file:
```bash
sed -i 's/geant4_set_and_check_package_variable(EXPAT_LIBRARY ""  "")/geant4_set_and_check_package_variable(EXPAT_LIBRARY "" FILEPATH "Path to a library.")/' \
  /media/uadm/data5/GAMPix4SNe/opt/geant4-v10.7.4/lib/Geant4-10.7.4/Geant4PackageCache.cmake
```

---

### 5. edep-sim cmake fails: XercesC version not found

**Symptom:**
```
Failed to find XercesC (missing: XercesC_VERSION)
```

**Cause:** Passing `-DXercesC_INCLUDE_DIR=/usr/include/xercesc` (from macOS instructions) conflicts
with what Geant4 recorded at build time. On Ubuntu the correct include root is `/usr/include`,
not the `xercesc/` subdirectory.

**Fix:** Remove the manual XercesC flags entirely. With `libxerces-c-dev` installed system-wide,
cmake finds it automatically:
```bash
# Do NOT pass -DXercesC_INCLUDE_DIR or -DXercesC_LIBRARY_RELEASE on Ubuntu
cmake \
  -DCMAKE_INSTALL_PREFIX=... \
  /path/to/edep-sim-origin
```

---

### 6. edep-sim build fails: `std::vector` not declared

**Symptom:**
```
error: 'std::vector' has not been declared
  int GetTokens(std::vector<std::string>& tokens);
```

**Cause:** `EDepSimHEPEVTCountGenerator.hh` is missing `#include <vector>`. GCC 9 with `-std=c++11`
does not pull this in implicitly.

**Fix:** Patch the header:
```bash
sed -i '/#include <fstream>/a #include <vector>' \
  /media/uadm/data5/GAMPix4SNe/edep-sim-origin/src/kinem/EDepSimHEPEVTCountGenerator.hh
```

Then rebuild without re-running cmake:
```bash
cd /media/uadm/data5/GAMPix4SNe/build/edep-sim
make -j$(nproc)
```
