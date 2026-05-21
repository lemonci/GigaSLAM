# GigaSLAM Install Notes

Setup notes for installing GigaSLAM into a conda env on a system where the host CUDA toolkit is newer than what the project requires. Supplements `README.md`.

## Tested target

- Host system: Ubuntu 24.04 derivative, kernel 6.17, system CUDA 13.0, system GCC 13.3
- GPU: NVIDIA RTX 5000 Ada Generation (compute capability **8.9** / sm_89)
- All toolchain pulled into the `gigaslam` conda env — no `sudo`, no touching system packages.

## Final versions installed

| Component | Version | Source |
| --- | --- | --- |
| Python | 3.10.20 (CPython) | conda-forge |
| CUDA toolkit + nvcc | 11.8.89 | `nvidia/label/cuda-11.8.0` |
| GCC / G++ | 11.4.0 | conda-forge (`gcc_linux-64`, `gxx_linux-64`) |
| CMake | 4.3.2 | conda-forge |
| OpenCV C++ | 4.13.0 | conda-forge `libopencv` |
| PyTorch | 2.2.0+cu118 | download.pytorch.org/whl/cu118 |
| xformers | 0.0.24+**cu118** | download.pytorch.org/whl/cu118 |
| torch_scatter | 2.1.2+pt22cu118 | data.pyg.org |
| simple_knn, diff_gaussian_rasterization, dpretrieval, sim3solve | local source | compiled for sm_89 |
| DBoW2 C++ lib | installed to `$CONDA_PREFIX/lib` | local source |

## Install order

```bash
# 0. Create env (if not already)
conda create -n gigaslam python=3.10
conda activate gigaslam

# 1. CUDA 11.8 toolkit into the env
conda install -y -c nvidia/label/cuda-11.8.0 cuda-toolkit

# 2. GCC 11, cmake, OpenCV C++ into the env (note: forces cpython, see Gotcha 1)
conda install -y -c conda-forge \
    "python=3.10.*=*cpython*" \
    cmake "gcc_linux-64=11.*" "gxx_linux-64=11.*" sysroot_linux-64 \
    "libopencv=4.*" pkg-config

# 3. PyTorch 2.2.0 + cu118
pip install torch==2.2.0 torchvision==0.17.0 torchaudio==2.2.0 \
    --index-url https://download.pytorch.org/whl/cu118

# 4. torch_scatter prebuilt wheel (avoid source build)
pip install https://data.pyg.org/whl/torch-2.2.0%2Bcu118/torch_scatter-2.1.2%2Bpt22cu118-cp310-cp310-linux_x86_64.whl

# 5. Project Python deps
pip install -r requirements.txt

# 6. Replace PyPI xformers (cu121-built) with cu118-built wheel — see Gotcha 2
pip install --force-reinstall --no-deps xformers==0.0.24+cu118 \
    --index-url https://download.pytorch.org/whl/cu118

# 7. 3DGS CUDA modules (see Gotcha 3 about --no-build-isolation)
export TORCH_CUDA_ARCH_LIST="8.9"   # for RTX 5000 Ada
pip install --no-build-isolation ./submodules/simple-knn
pip install --no-build-isolation ./submodules/diff-gaussian-rasterization

# 8. DBoW2 → install into the conda prefix (no sudo) — see Gotcha 4
cd DBoW2 && rm -rf build && mkdir build && cd build
cmake .. -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
         -DCMAKE_INSTALL_PREFIX=$CONDA_PREFIX \
         -DBUILD_Demo=OFF
make -j$(nproc) && make install
cd ../..

# 9. DPRetrieval (picks up DBoW2 from $CONDA_PREFIX automatically)
export CMAKE_ARGS="$CMAKE_ARGS -DCMAKE_POLICY_VERSION_MINIMUM=3.5"
pip install --no-build-isolation ./DPRetrieval

# 10. sim3solve (loop closure correction)
python setup.py install

# 11. ORB vocabulary
wget https://github.com/UZ-SLAMLab/ORB_SLAM3/raw/master/Vocabulary/ORBvoc.txt.tar.gz
tar -xzf ORBvoc.txt.tar.gz && rm ORBvoc.txt.tar.gz
```

## Gotchas (the parts not in the README)

### 1. conda-forge can swap CPython for GraalPy when installing OpenCV
When `libopencv` is resolved from conda-forge in a fresh env, the solver may pull `graalpy 23.0.0` as the `python 3.10` provider. PyTorch/pip wheels won't work on GraalPy. **Pin to CPython explicitly:**
```bash
conda install "python=3.10.*=*cpython*"
```

### 2. PyPI `xformers==0.0.24` is built against CUDA 12.1
On a cu118 PyTorch, importing it prints:
```
WARNING[XFORMERS]: xFormers can't load C++/CUDA extensions.
  PyTorch 2.2.0+cu121 with CUDA 1201 (you have 2.2.0+cu118)
  Memory-efficient attention, SwiGLU, sparse and more won't be available.
```
The Python-level API still works, but CUDA kernels are disabled. Replace with the cu118-built wheel from pytorch.org:
```bash
pip install --force-reinstall --no-deps xformers==0.0.24+cu118 \
    --index-url https://download.pytorch.org/whl/cu118
```

### 3. `pip install ./submodules/...` fails with `ModuleNotFoundError: No module named 'torch'`
Modern pip uses build isolation, which hides `torch` from `setup.py` even though it's installed in the env. Add `--no-build-isolation` for any setup.py that does `from torch.utils.cpp_extension import CUDAExtension`.

### 4. CMake 4.x rejects `cmake_minimum_required(VERSION 3.0)`
DBoW2's `CMakeLists.txt` declares a minimum of 3.0, which CMake ≥ 4.0 refuses. Pass `-DCMAKE_POLICY_VERSION_MINIMUM=3.5` (or downgrade cmake to 3.x). Same flag needed for `DPRetrieval` — propagate it via `CMAKE_ARGS` since DPRetrieval's setup.py drives cmake internally.

### 5. Install DBoW2 to `$CONDA_PREFIX`, not system-wide
The README says `sudo make install`, which writes to `/usr/local`. Instead, use `-DCMAKE_INSTALL_PREFIX=$CONDA_PREFIX` so it lives inside the conda env and DPRetrieval's `find_package(DBoW2)` picks it up via `CMAKE_PREFIX_PATH` (conda-forge sets this for you on activate). Removing the env then removes DBoW2 too.

### 6. Set `TORCH_CUDA_ARCH_LIST` for newer GPUs
PyTorch 2.2.0 binary ships kernels for sm_50…86 and sm_90 (no sm_89). The auto-detect is usually right, but on Ada (RTX 4090 / RTX 5000 Ada) it's safer to set it explicitly before compiling any CUDA extension:
```bash
export TORCH_CUDA_ARCH_LIST="8.9"
```

## Verifying the install

```bash
conda activate gigaslam
python - <<'EOF'
import torch, xformers, xformers.ops, torch_scatter
import simple_knn._C, diff_gaussian_rasterization, dpretrieval, sim3solve
print("torch:", torch.__version__, "| cuda:", torch.cuda.is_available(), "| dev:", torch.cuda.get_device_name(0))
print("xformers:", xformers.__version__)
print("torch_scatter:", torch_scatter.__version__)
# xformers CUDA kernel smoke test
q = torch.randn(1, 8, 64, 32, device='cuda', dtype=torch.float16)
xformers.ops.memory_efficient_attention(q, q, q)
print("xformers cuda kernels: OK")
EOF
```

Expected output includes `cuda: True`, `xformers: 0.0.24+cu118`, and no warning about disabled C++/CUDA extensions.

## Running

```bash
conda activate gigaslam
python slam.py --config ./configs/kitti_06.yaml
```

Edit the chosen `configs/*.yaml` first to set `Dataset.color_path` and the camera intrinsics. Pretrained DISK / LightGlue / UniDepth weights auto-download on first run; see the README for the HuggingFace mirror workaround and the pinned UniDepthV2 weights link.
