# Building Ollama on IBM AC922 (RHEL 8.10)

Complete guide for compiling and installing Ollama with CUDA GPU support on IBM Power AC922 systems running RHEL 8.10.

**Build Information:**
- **Base Version**: Latest Ollama main branch (development)
- **AC922 Branch**: Includes Power architecture patches
- **Commit**: 7a4d04f1 (Apply Power architecture patches for ppc64le support)
- **Upstream Sync**: Merged with ollama:main (commit f8dc7c9f)

## System Requirements

- **Hardware**: IBM Power AC922 with NVIDIA V100 GPUs
- **OS**: RHEL 8.10 (ppc64le)
- **Architecture**: Power9
- **GPUs**: NVIDIA Tesla V100 (tested with 4x V100-SXM2-16GB)

## Prerequisites

### 1. Install CUDA Toolkit (if not already installed)

Download and install CUDA 11.8 for ppc64le:

**Download Link**: [CUDA 11.8.0 for RHEL 8 ppc64le](https://developer.nvidia.com/cuda-11-8-0-download-archive?target_os=Linux&target_arch=ppc64le&Distribution=RHEL&target_version=8&target_type=rpm_local)

```bash
# Download the installer
wget https://developer.download.nvidia.com/compute/cuda/11.8.0/local_installers/cuda-repo-rhel8-11-8-local-11.8.0_520.61.05-1.ppc64le.rpm

# Install the repository
sudo rpm -i cuda-repo-rhel8-11-8-local-11.8.0_520.61.05-1.ppc64le.rpm

# Clean and install CUDA
sudo dnf clean all
sudo dnf -y install cuda-toolkit-11-8

# Verify installation
nvidia-smi
ls -la /usr/local/cuda-11.8/lib64/libcudart.so*
```

**Note**: This guide is tested with CUDA 11.8. While newer CUDA versions may work, CUDA 11.8 is the verified configuration for this build.

### 2. Verify GPU Detection

```bash
# Check NVIDIA GPUs are detected
nvidia-smi

# Expected output: List of V100 GPUs with driver version
```

### 3. Install GCC Toolset 11

CUDA 11.8 requires GCC ≤ 11:

```bash
# Install GCC 11
sudo yum install gcc-toolset-11

# Verify installation
ls /opt/rh/gcc-toolset-11/root/usr/bin/gcc
```

### 4. Install Go 1.24.1 for ppc64le

**Download Link**: [Go 1.24.1 for Linux ppc64le](https://go.dev/dl/go1.24.1.linux-ppc64le.tar.gz)

```bash
# Download Go for Power architecture
cd /tmp
wget https://go.dev/dl/go1.24.1.linux-ppc64le.tar.gz

# Extract to a permanent location
tar xzf go1.24.1.linux-ppc64le.tar.gz
sudo mv go /usr/local/go-1.24.1

# Add to PATH (add to ~/.bashrc for persistence)
export PATH=/usr/local/go-1.24.1/bin:$PATH

# Verify installation
go version
# Expected: go version go1.24.1 linux/ppc64le
```

**Alternative Go versions**: You can also use Go 1.23.x or later from [https://go.dev/dl/](https://go.dev/dl/)

### 5. Install CMake (if not present)

```bash
sudo yum install cmake
cmake --version  # Should be 3.x or higher
```

## Building Ollama with CUDA Support

### Step 1: Clone the Repository

```bash
# Clone the patched repository with Power architecture support
git clone -b AC922 https://github.com/naveedus/ollama-ppc64le.git
cd ollama-ppc64le

# Set a convenient base directory variable
export OLLAMA_DIR=$(pwd)
```

**Note**: The AC922 branch includes patches for Power architecture support:
- Power9/Power10 CPU optimizations
- CUDA support for ppc64le
- Vector instruction support (VSX)

### Step 2: Setup Build Environment

```bash
# Enable GCC 11 toolset (choose one method)
# Method 1: Using scl enable
scl enable gcc-toolset-11 bash

# Method 2: Using export (if scl not available)
export PATH=/opt/rh/gcc-toolset-11/root/usr/bin/:$PATH
source scl_source enable gcc-toolset-11

# Add CUDA to PATH
export PATH=/usr/local/cuda-11.8/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-11.8/lib64:$LD_LIBRARY_PATH

# Add Go to PATH
export PATH=/usr/local/go-1.24.1/bin:$PATH

# Verify all tools are accessible
gcc --version    # Should show 11.x
nvcc --version   # Should show CUDA 11.8.89
go version       # Should show go1.24.1

# Make environment persistent (optional but recommended)
cat >> ~/.bashrc << 'EOF'
# Ollama build environment
export PATH=/opt/rh/gcc-toolset-11/root/usr/bin/:$PATH
export PATH=/usr/local/cuda-11.8/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-11.8/lib64:$LD_LIBRARY_PATH
export PATH=/usr/local/go-1.24.1/bin:$PATH
EOF

source ~/.bashrc
```

### Step 3: Build Native Libraries with CMake

```bash
# Configure build for V100 GPUs (compute capability 7.0)
cmake -B build \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda-11.8/bin/nvcc \
  -DCMAKE_CUDA_ARCHITECTURES=70 \
  -DCMAKE_C_COMPILER=/opt/rh/gcc-toolset-11/root/usr/bin/gcc \
  -DCMAKE_CXX_COMPILER=/opt/rh/gcc-toolset-11/root/usr/bin/g++

# Build (this takes 10-20 minutes)
cmake --build build -j$(nproc)
```

**Note**: Power10 backend may fail to compile - this is expected and doesn't affect functionality on AC922 (Power9).

### Step 4: Verify Libraries Were Built

```bash
# Check CUDA library
ls -lh build/lib/ollama/libggml-cuda.so
# Expected: ~40-50MB file

# Check Power9 CPU library
ls -lh build/lib/ollama/libggml-cpu-power9.so
# Expected: ~5-10MB file
```

### Step 5: Build Ollama Binary

```bash
# Set library paths for Go build
export CGO_LDFLAGS="-L$(pwd)/build/lib/ollama/ -lggml-cuda -lggml-cpu-power9"
export CGO_CFLAGS="-I$(pwd)/ml/backend/ggml/ggml/include"

# Build Ollama for Power9 architecture
go build --tags ppc64le.power9 -o ollama .
```

**Build time**: 5-15 minutes depending on system

### Step 6: Verify Build

```bash
# Check binary was created
ls -lh ollama
# Expected: ~100-200MB executable

# Test basic functionality
./ollama --version
# Expected output: ollama version is 0.0.0
```

**Note about version**: Development builds show version `0.0.0`. This is normal and doesn't affect functionality. To check the actual upstream version:

```bash
# Check git commit information
git log --oneline -1
# Shows: 7a4d04f1 Apply Power architecture patches for ppc64le support

# Check upstream sync status
git log --oneline | head -5
# Shows recent commits merged from ollama:main
```

**Optional: Set a custom version**

If you want a proper version number:

```bash
# Create version file
mkdir -p version
echo "0.15.6-ac922-$(date +%Y%m%d)" > version/version.txt

# Rebuild
go build --tags ppc64le.power9 -o ollama .

# Check version
./ollama --version
# Now shows: ollama version is 0.15.6-ac922-20260212
```

## Running Ollama

### Start Ollama Server

```bash
# Set runtime library paths
export LD_LIBRARY_PATH=$OLLAMA_DIR/build/lib/ollama:/usr/local/cuda-11.8/lib64:$LD_LIBRARY_PATH

# Enable all GPUs (or specify subset: 0,1 for first two)
export CUDA_VISIBLE_DEVICES=0,1,2,3

# Start server
export OLLAMA_HOST=0.0.0.0:11434
./ollama serve
```

**Permanent LD_LIBRARY_PATH Setup:**

To avoid setting LD_LIBRARY_PATH in every terminal:

```bash
# Add to ~/.bashrc
echo "export LD_LIBRARY_PATH=$OLLAMA_DIR/build/lib/ollama:/usr/local/cuda-11.8/lib64:\$LD_LIBRARY_PATH" >> ~/.bashrc
source ~/.bashrc
```

**Verify GPU is Being Used:**

```bash
# Check which libraries Ollama is linked against
ldd ./ollama | grep cuda

# Expected output should show:
# libggml-cuda.so => /path/to/build/lib/ollama/libggml-cuda.so
# libcudart.so.11.0 => /usr/local/cuda-11.8/lib64/libcudart.so.11.0
# libcublas.so.11 => /usr/local/cuda-11.8/lib64/libcublas.so.11
```

**Expected output** (confirming GPU detection):
```
time=... level=INFO msg="inference compute" id=GPU-0 library=cuda compute="7.0" name="Tesla V100-SXM2-16GB" total="16.0 GiB"
time=... level=INFO msg="inference compute" id=GPU-1 library=cuda compute="7.0" name="Tesla V100-SXM2-16GB" total="16.0 GiB"
time=... level=INFO msg="inference compute" id=GPU-2 library=cuda compute="7.0" name="Tesla V100-SXM2-16GB" total="16.0 GiB"
time=... level=INFO msg="inference compute" id=GPU-3 library=cuda compute="7.0" name="Tesla V100-SXM2-16GB" total="16.0 GiB"
```

### Download and Run a Model

In a new terminal:

```bash
# Set library path (if not in ~/.bashrc)
cd /path/to/ollama-ppc64le
export LD_LIBRARY_PATH=$(pwd)/build/lib/ollama:/usr/local/cuda-11.8/lib64:$LD_LIBRARY_PATH

# Pull a model (first time only)
./ollama pull llama3.2:3b

# Run the model
./ollama run llama3.2:3b "Explain quantum computing in simple terms"
```

### Monitor GPU Usage

```bash
# Watch GPU utilization in real-time
watch -n 1 nvidia-smi
```

You should see GPU memory usage and utilization increase during inference.

## Remote Access

### From Another Machine

```bash
# Set Ollama host on remote machine
export OLLAMA_HOST=http://<AC922_IP>:11434

# Use Ollama normally
ollama list
ollama run llama3.2:3b "Hello!"
```

### Using API

```bash
# From any machine
curl http://<AC922_IP>:11434/api/generate -d '{
  "model": "llama3.2:3b",
  "prompt": "Why is the sky blue?",
  "stream": false
}'
```

## Systemd Service (Optional)

Create a systemd service for automatic startup:

```bash
# Replace /path/to/ollama-ppc64le with your actual installation path
sudo tee /etc/systemd/system/ollama.service > /dev/null <<EOF
[Unit]
Description=Ollama Service
After=network.target

[Service]
Type=simple
User=ollama
WorkingDirectory=/path/to/ollama-ppc64le
Environment="LD_LIBRARY_PATH=/path/to/ollama-ppc64le/build/lib/ollama:/usr/local/cuda-11.8/lib64"
Environment="CUDA_VISIBLE_DEVICES=0,1,2,3"
Environment="OLLAMA_HOST=0.0.0.0:11434"
ExecStart=/path/to/ollama-ppc64le/ollama serve
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

# Note: Replace 'ollama' user with your preferred service account
# For root user, change User=ollama to User=root

# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable ollama
sudo systemctl start ollama

# Check status
sudo systemctl status ollama
```

### Firewall Configuration

If you need to access Ollama from other machines:

```bash
# Open port 11434 in firewall
sudo firewall-cmd --permanent --add-port=11434/tcp
sudo firewall-cmd --reload

# Verify port is open
sudo firewall-cmd --list-ports
```

### SELinux Considerations

If SELinux is enforcing and you encounter permission issues:

```bash
# Check SELinux status
getenforce

# If Enforcing, you may need to allow the port
sudo semanage port -a -t http_port_t -p tcp 11434

# Or temporarily set to permissive for testing
sudo setenforce 0

# To make permissive permanent (not recommended for production)
# sudo sed -i 's/SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

## Troubleshooting

### CUDA Not Detected

If GPUs aren't detected:

1. **Check NVIDIA drivers**:
   ```bash
   nvidia-smi
   ```

2. **Verify CUDA libraries are in path**:
   ```bash
   echo $LD_LIBRARY_PATH
   ldconfig -p | grep cuda
   ```

3. **Check Ollama logs**:
   ```bash
   export OLLAMA_DEBUG=1
   ./ollama serve
   ```

### Build Errors

**Power10 compilation fails**: This is expected on AC922 (Power9). The build will succeed for Power9 and CUDA backends.

**GCC version error**: Ensure you're using GCC 11:
```bash
gcc --version  # Must show 11.x
which gcc      # Should point to /opt/rh/gcc-toolset-11/...
```

**Go not found**: Verify Go is in PATH:
```bash
go version
export PATH=/usr/local/go-1.24.1/bin:$PATH
```

### Performance Issues

1. **Check GPU utilization**:
   ```bash
   nvidia-smi dmon
   ```

2. **Verify CUDA is being used**:
   Look for "library=cuda" in Ollama logs

3. **Adjust thread count**:
   ```bash
   export OLLAMA_NUM_THREADS=32
   ```

## Performance Notes

- **AC922 Specs**: 4x V100-SXM2-16GB = 64GB total GPU memory
- **NVLink**: AC922 has NVLink 2.0 for fast multi-GPU communication
- **Recommended models**:
  - Small: llama3.2:1b, llama3.2:3b
  - Medium: llama3.1:8b, mistral:7b
  - Large: llama3.1:70b (requires multiple GPUs)

**Note**: This guide focuses on GPU-only inference. CPU fallback has not been extensively tested.

## Download Links

- **CUDA Toolkit**: [CUDA 11.8.0 for RHEL 8 ppc64le](https://developer.nvidia.com/cuda-11-8-0-download-archive?target_os=Linux&target_arch=ppc64le&Distribution=RHEL&target_version=8&target_type=rpm_local)
- **Go Language**: [Go Downloads](https://go.dev/dl/) - Select linux-ppc64le version
- **Ollama Models**: [Ollama Model Library](https://ollama.com/library)

## Additional Resources

- [Ollama Documentation](https://github.com/ollama/ollama)
- [NVIDIA CUDA Documentation](https://docs.nvidia.com/cuda/)
- [IBM Power Systems](https://www.ibm.com/power)
- [Go Programming Language](https://go.dev/)

## Credits

- Patches for Power architecture support from:
  - https://github.com/ollama/ollama/commit/4115e4f58f8c3fb86cdce2d58aae811c2d26cc52
  - https://github.com/ollama/ollama/pull/13972

## License

Same as upstream Ollama project (MIT License)

---

**Signed-off-by**: NaveedUS, 2026
**Documentation**: Assisted with AI tools