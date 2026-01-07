# Complete installation of: AutoCAS and QCMaquis

**Updated On:** January 2026
___
## TOC
1. [Directory Structure Setup](#directory-structure-setup)
2. [QCMaquis Installation](#qcmaquis-installation)
3. [AutoCAS Installation](#autocas-installation)
4. [Module File Creation](#module-file-creation)
5. [Testing](#testing)

## Directory Structure Setup

### Step 1: Create Installation Directories
```bash
# Create organized directory structure
mkdir -p ~/software/{source,build,install}
mkdir -p ~/software/source/{qcmaquis,autocas}
mkdir -p ~/software/build/{qcmaquis,autocas}
mkdir -p ~/software/install/{qcmaquis,autocas}
mkdir -p ~/privatemodules/{qcmaquis,autocas}

# Take a look to verify
ls -la ~/software/
ls -la ~/privatemodules/
```
---

## QCMaquis Installation

### Step 2: Create Module Loading Script
```bash
# Create script to load all required modules
cat > ~/software/source/qcmaquis/load_modules.sh << 'SCRIPT_END'
#!/bin/bash
# Module loading script for QCMaquis and AutoCAS installation
# Toolchain: gcc/12.3 with compatible dependencies

echo "Loading modules for QCMaquis/AutoCAS build..."

module purge
module load StdEnv/2023
module load gcc/12.3
module load openmpi/4.1.5
module load cmake/3.31.0
module load boost/1.85.0
module load hdf5/1.14.2
module load gsl/2.7
module load eigen/3.4.0

echo ""
echo "Modules loaded successfully!"
echo "============================="
module list
echo "============================="
echo ""

# Display important environment variables
echo "Key Environment Variables:"
echo "EBROOTGCC = $EBROOTGCC"
echo "EBROOTBOOST = $EBROOTBOOST"
echo "EBROOTHDF5 = $EBROOTHDF5"
echo "EBROOTGSL = $EBROOTGSL"
echo "EBROOTEIGEN = $EBROOTEIGEN"
echo ""
echo "Compilers:"
echo "CC  = $(which gcc)"
echo "CXX = $(which g++)"
echo "CMAKE = $(which cmake)"
SCRIPT_END

chmod +x ~/software/source/qcmaquis/load_modules.sh
```

### Step 3: Test Module Loading
```bash
source ~/software/source/qcmaquis/load_modules.sh
```
**You should see:** All modules loading without errors, with EBROOT variables set.

### Step 4: Download QCMaquis Source
```bash
cd ~/software/source/qcmaquis
git clone https://github.com/qcscine/qcmaquis.git
cd qcmaquis

# Checkout stable release
git checkout v4.0.0

# Initialize submodules (includes ALPS library)
git submodule update --init --recursive

# Verify download
ls -la
git describe --tags
```
**You should see:** 'v4.0.0' and directory with source files.
### Step 5: Configure QCMaquis Build
```bash
# Go to the build directory
cd ~/software/build/qcmaquis

# Load modules
source ~/software/source/qcmaquis/load_modules.sh

# Configure with CMake
cmake \
  -DCMAKE_INSTALL_PREFIX=$HOME/software/install/qcmaquis \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CXX_COMPILER=g++ \
  -DCMAKE_C_COMPILER=gcc \
  -DBLAS_LAPACK_SELECTOR=auto \
  -DBOOST_ROOT=$EBROOTBOOST \
  -DHDF5_ROOT=$EBROOTHDF5 \
  -DGSL_ROOT_DIR=$EBROOTGSL \
  -DEigen3_DIR=$EBROOTEIGEN \
  -DBUILD_MANUAL=OFF \
  -DQCMAQUIS_TESTS=ON \
  $HOME/software/source/qcmaquis/qcmaquis
```

**You should see:** "Configuring done" and "Generating done" messages.

**Important CMake Options:**
- `Release` build type for optimized performance
- `TESTS=ON` to enable validation
- Uses EBROOT variables for library paths (set by modules)

### Step 6: Compile QCMaquis
```bash
# Build with 8 parallel jobs (adjust based on available cores)
make -j 8
```

**Notes:** This will take a bit of time, and you should see progress eventually reach 100% with no fatal errors. Some warnings are normal and can be ignored. 


### Step 7: Run Tests
```bash
# Run test suite
ctest --output-on-failure
```

**Notes:** You should see 100% of tests pass. If not, try reruning (maybe sensative to system load?)

### Step 8: Install QCMaquis
```bash
make install

# Check for installation
ls -la ~/software/install/qcmaquis/
ls -la ~/software/install/qcmaquis/lib/libmaquis_dmrg.so
```


### Step 9: Install QCMaquis Python Package
```bash
# Load Python module
module load python/3.12.4

# Create virtual environment
cd ~/software/install/qcmaquis
virtualenv --no-download qcmaquis_env

# Activate environment
source ~/software/install/qcmaquis/qcmaquis_env/bin/activate

# Upgrade pip
pip install --no-index --upgrade pip

# Install setuptools
pip install --no-index setuptools wheel

# Go to the source and install
cd ~/software/source/qcmaquis/qcmaquis
pip install --no-build-isolation -e .
```

**Notes:** You should see "Sucessfully installed scine_qcmaquis-4.0.0". As well, pip may downgrade numpy, this is ok.

### Step 10: Test QCMaquis
```bash
# Add library to path
export LD_LIBRARY_PATH=$HOME/software/install/qcmaquis/lib:$LD_LIBRARY_PATH

# Test command-line tool
qcmaquis --version
```
**Notes:** Here you should see:
```
SCINE QCMaquis
Quantum Chemical Density Matrix Renormalization group
DMRG version 3.1.3-... v4.0.0
```
## AutoCAS Installation
### Step 11: Download AutoCAS
```bash
cd ~/software/source/autocas
git clone https://github.com/qcscine/autocas.git .

# Checkout stable release
git checkout 3.0.0

# Initialize submodules
git submodule update --init --recursive

# Verify
ls -la
```
### Step 12: Create AutoCAS Virtual Environment
```bash
# Load modules
module purge
module use ~/privatemodules  # We'll create this module next
module load python/3.12.4

# Create virtual environment
cd ~/software/install/autocas
virtualenv --no-download autocas_env

# Activate
source ~/software/install/autocas/autocas_env/bin/activate

# Upgrade pip
pip install --no-index --upgrade pip
```

### Step 13: Install AutoCAS Dependencies
```bash
# Install dependencies with flexible versions
pip install --no-index \
  cycler \
  fonttools \
  h5py \
  kiwisolver \
  matplotlib \
  numpy \
  packaging \
  Pillow \
  pyparsing \
  python-dateutil \
  PyYAML \
  six \
  mypy-extensions \
  setuptools \
  wheel
```

### Step 14: Modify AutoCAS Setup for Alliance Wheelhouse

The AutoCAS setup.py has strict version pinning that conflicts with Alliance wheelhouse versions. Modifying as followed seemed to fix any problems:
```bash
cd ~/software/source/autocas

# Backup original (if you want)
cp setup.py setup.py.original

# Create modified version
cat > setup.py << 'SETUP_END'
"""
autoCAS
Automated active space selection for multi-configurational calculations.
"""
from setuptools import setup, find_packages

# Read README file
def readme():
    with open('README.rst') as f:
        return f.read()

# Requirements with flexible versions for Alliance Canada wheelhouse
install_requires = [
    'cycler',
    'fonttools',
    'h5py>=3.0.0',
    'kiwisolver',
    'matplotlib>=3.0.0',
    'numpy>=1.20.0',
    'packaging',
    'Pillow',
    'pyparsing',
    'python-dateutil',
    'PyYAML',
    'six',
    'mypy-extensions',
]

setup(
    name='scine_autocas',
    version='3.0.0',
    url='https://github.com/qcscine/autocas',
    author='ETH Zurich, Laboratory of Physical Chemistry, Reiher Group',
    author_email='scine@phys.chem.ethz.ch',
    description='Automated active space selection',
    long_description=readme(),
    install_requires=install_requires,
    packages=find_packages(),
    include_package_data=True,
    classifiers=[
        'Development Status :: 4 - Beta',
        'Intended Audience :: Science/Research',
        'License :: OSI Approved :: BSD License',
        'Natural Language :: English',
        'Programming Language :: Python :: 3',
    ],
)
SETUP_END
```


### Step 15: Install AutoCAS
```bash
# Make sure we're in the autocas environment
source ~/software/install/autocas/autocas_env/bin/activate

# Also ensure QCMaquis is in the Python path
export PYTHONPATH=$HOME/software/install/qcmaquis/qcmaquis_env/lib/python3.12/site-packages:$PYTHONPATH

# Install AutoCAS
cd ~/software/source/autocas
pip install --no-build-isolation -e .
```
**Notes:** You should see: "successfully installed scine_autocas-3.0.0"

### Step 16: Test AutoCAS
```bash
# Test import
python -c "import scine_autocas; print('AutoCAS imported successfully')"

# Test interface
python -c "from scine_autocas.interfaces import molcas; print('AutoCAS interfaces loaded')"
```

**Notes:** Here, you should see them both complete without errors.

## Module File Creation
Doing this allows you to load these like any other module currently on the system! 

### Step 17: Create QCMaquis Module
```bash
cat > ~/privatemodules/qcmaquis/4.0.0.lua << 'MODULE_END'
help([[
SCINE QCMaquis - DMRG quantum chemistry software
Version 4.0.0
]])

whatis("Name: QCMaquis")
whatis("Version: 4.0.0")
whatis("Category: Quantum Chemistry")
whatis("Description: DMRG software for quantum chemical Hamiltonians")
whatis("URL: https://github.com/qcscine/qcmaquis")

-- Load dependencies
depends_on("StdEnv/2023")
depends_on("gcc/12.3")
depends_on("openmpi/4.1.5")
depends_on("boost/1.85.0")
depends_on("hdf5/1.14.2")
depends_on("gsl/2.7")
depends_on("eigen/3.4.0")
depends_on("python/3.12.4")

local base = pathJoin(os.getenv("HOME"), "software/install/qcmaquis")
local venv = pathJoin(base, "qcmaquis_env")

-- Add library paths
prepend_path("LD_LIBRARY_PATH", pathJoin(base, "lib"))
prepend_path("LIBRARY_PATH", pathJoin(base, "lib"))
prepend_path("CPATH", pathJoin(base, "include"))

-- Add binaries
prepend_path("PATH", pathJoin(base, "bin"))
prepend_path("PATH", pathJoin(venv, "bin"))

-- Python paths
prepend_path("PYTHONPATH", pathJoin(venv, "lib/python3.12/site-packages"))

-- Environment variables
setenv("QCMAQUIS_ROOT", base)
setenv("VIRTUAL_ENV", venv)

-- CMake integration
prepend_path("CMAKE_PREFIX_PATH", base)
MODULE_END
```

### Step 18: Create AutoCAS Module
```bash
cat > ~/privatemodules/autocas/3.0.0.lua << 'MODULE_END'
help([[
SCINE AutoCAS - Automated active orbital space selection
Version 3.0.0
Requires QCMaquis 4.0.0
]])

whatis("Name: AutoCAS")
whatis("Version: 3.0.0")
whatis("Category: Quantum Chemistry")
whatis("Description: Automated active orbital space selection for multi-configurational calculations")
whatis("URL: https://github.com/qcscine/autocas")

-- Load QCMaquis (AutoCAS depends on it)
load("qcmaquis/4.0.0")

local base = pathJoin(os.getenv("HOME"), "software/install/autocas")
local venv = pathJoin(base, "autocas_env")

-- Add Python paths for AutoCAS
prepend_path("PATH", pathJoin(venv, "bin"))
prepend_path("PYTHONPATH", pathJoin(venv, "lib/python3.12/site-packages"))

-- Set environment variables
setenv("AUTOCAS_ROOT", base)
setenv("AUTOCAS_VENV", venv)
MODULE_END
```

---

## Testing

### Step 19: Test Module Loading
```bash
# Enable private modules
module use ~/privatemodules

# Test QCMaquis module
module purge
module load qcmaquis/4.0.0
module list
qcmaquis --version
python -c "import scine_qcmaquis; print('QCMaquis OK')"

# Test AutoCAS module
module purge
module load autocas/3.0.0
module list
python -c "import scine_autocas; print('AutoCAS OK')"
```
**Notes:** Both modules should load fine at this point! You will have to run the `module use ~/privatemodules` before loading either of these. OR: `echo 'module use ~/privatemodules' >> ~/.bashrc` to load it every session automatically!








