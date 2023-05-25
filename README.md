# pytorch_rocm_notes
Notes on building pytorch on a fresh ubuntu 22.04 server install for Radeon RX 5700 XT

Stealing time on my son's gaming rig from the beginning of the pandemic to test out some Machine Learning algorithms on the AMD GPU. Pretty much I want to get this working because it's more dormant now that he's got a steam deck.

Started out with a fresh Ubuntu 22.04.2 server install, needed to expand the hard drive (since this step was crashing the installer...)
```bash
sudo vgdisplay
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

Then update packages, install the GPU drivers, reboot
```bash
wget https://repo.radeon.com/amdgpu-install/5.5.1/ubuntu/jammy/amdgpu-install_5.5.50501-1_all.deb
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install ./amdgpu-install_5.5.50501-1_all.deb
sudo amdgpu-install --usecase="dkms,rocm"
sudo reboot
```

Install dependencies, set up virtual environment, clone git repos, install more requirements, set up build for AMD
```bash
sudo apt-get install python3.10-venv python3.10-dev libstdc++-12-dev
python3 -m venv pytorch-build
source pytorch-build/bin/activate
pip install cmake ninja
git clone --recursive https://github.com/pytorch/pytorch.git 
cd pytorch 
pip install -r requirements.txt
python tools/amd_build/build_amd.py
```
Fix a bug in the cmake build dependencies
```diff
diff --git a/cmake/Dependencies.cmake b/cmake/Dependencies.cmake
index 6138955342a..4bfce2a4f27 100644
--- a/cmake/Dependencies.cmake
+++ b/cmake/Dependencies.cmake
@@ -1309,7 +1309,7 @@ if(USE_ROCM)
     # If you get this wrong, you'll get a complaint like 'ld: cannot find -lrocblas-targets'
     if(ROCM_VERSION_DEV VERSION_GREATER_EQUAL "4.1.0")
       list(APPEND Caffe2_PUBLIC_HIP_DEPENDENCY_LIBS
-        roc::rocblas hip::hipfft hip::hiprand roc::hipsparse)
+        roc::rocblas hip::hipfft hip::hiprand roc::hipsparse ncurses)
     else()
       list(APPEND Caffe2_PUBLIC_HIP_DEPENDENCY_LIBS
         roc::rocblas roc::rocfft hip::hiprand roc::hipsparse)
```

Build pytorch (takes a while)
```bash
PYTORCH_ROCM_ARCH=gfx1010 python setup.py bdist_wheel
```
