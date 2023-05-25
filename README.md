# pytorch_rocm_notes
Notes on building pytorch on a fresh ubuntu 22.04 server install for Radeon RX 5700 XT (currently flawed)

Stealing time on my son's gaming rig from the beginning of the pandemic to test out some Machine Learning algorithms on the AMD GPU. Pretty much I want to get this working because it's more dormant now that he's got a steam deck.

These instructions were completed with pytorch commit 4882cd08013733a5dbe299871ad7e974bce074b3 and torchvision commit 4125d3a02b15faf4b19767a91797320151ce8bc6

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
sudo usermod -a -G video $LOGNAME
sudo usermod -a -G render $LOGNAME
sudo reboot
```

Install dependencies, set up virtual environment, clone git repos, install more requirements, set up build for AMD
```bash
sudo apt-get install python3.10-venv python3.10-dev libstdc++-12-dev libjpeg-dev libpng-dev
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

Copy the wheel file off and install it for building torchvision
```bash
cp dist/torch-2.1.0a0+git4882cd0-cp310-cp310-linux_x86_64.whl ~
pip install dist/torch-2.1.0a0+git4882cd0-cp310-cp310-linux_x86_64.whl 
```
Clone torchvision
```bash
cd ..
git clone https://github.com/pytorch/vision.git
cd vision
PYTORCH_ROCM_ARCH=gfx1010 python setup.py bdist_wheel
cp dist/torchvision-0.16.0a0+4125d3a-cp310-cp310-linux_x86_64.whl ..
cd ..
deactivate
```

Our target test application is Stable Diffusion, specifically the Stable Diffusion Web UI
```bash
git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git
cd stable-diffusion-webui
```

Edit to add the following lines to webui-user.sh
```bash
export COMMANDLINE_ARGS="--api --listen"
export TORCH_COMMAND="pip install ~/torch-2.1.0a0+git4882cd0-cp310-cp310-linux_x86_64.whl ~/torchvision-0.16.0a0+4125d3a-cp310-cp310-linux_x86_64.whl"
```

Then start the webui
```bash
bash webui.sh
```

After that it dumps core after cross attention optimization so I'm still working on making this a functioning howto.
```
LatentDiffusion: Running in eps-prediction mode
DiffusionWrapper has 859.52 M params.
Applying cross attention optimization (Doggettx).
Segmentation fault (core dumped)
```
