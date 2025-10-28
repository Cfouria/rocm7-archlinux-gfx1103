# rocm7-archlinux-gfx1103
Just notes on how I built rocm-gfx1103.  
Encountered systemd-nspawn and restarted my attempts at ROCM. 

#  total    ~230GB LOTS of disk space usage  
THEROCK     ~120GB  ( git clone --depth 1 --revision ${_commit} ... bless)  
uv cache    ~30GB  
container   ~35GB   (when building wheel packages from THEROCK, 50GB is reached and ~15GB is 'cleaned up' as wheels are finished being made)  
/var        ~3GB  
/usr        ~5GB    
ccache      ~5GB  
comgr       ~5GB    (code object manager ~MIopen)  
sdnext      ~15GB  


# This is as I did it, sources are NOT in container so they are only downloaded once.
```
USER=c4
HOST=/home/$USER
CONTAINER=/opt/myContainer13

mkdir -p $HOST/ai && cd $HOST/ai
git clone --depth 1 -b dev https://github.com/vladmandic/sdnext
git clone --depth 1 -b main_perf https://github.com/ROCmSoftwarePlatform/flash-attention
git clone --depth 1 --shallow-submodules https://github.com/ROCm/TheRock
cd $HOST/ai/TheRock;
 git submodule foreach --recursive  git restore . ; git restore . ;
 git pull
 python ./build_tools/fetch_sources.py
 echo 'checkout default branch of everything EXCEPT 1.compiler 2.rocm-systems'
 echo 'compiler: triton blocks using up to date note: if you do checkout latest, it fails when launching sdnext, so way after many hours of compiling'
 echo 'rocm-systems: checkout latest blocked build with errors, saw was updated 2days ago and gave up messing with'
 function submodule-and-sub-submodule-default-brancher(){
  xargs -I{} sh -c 'cd {} ; git rev-parse --abbrev-ref origin/HEAD |  rg -v Enter | cut -c8- | xargs -n 1 git checkout && git pull ;
   git submodule foreach --recursive "git rev-parse --abbrev-ref origin/HEAD |  rg -v Enter | cut -c8- | xargs -n 1 git checkout && git pull "'
  }
git submodule status base \
  | awk -F ' ' '{print $2}' | submodule-and-sub-submodule-default-brancher
 git submodule status profiler \
  | awk -F ' ' '{print $2}' | submodule-and-sub-submodule-default-brancher
 git submodule status ml-libs \
  | awk -F ' ' '{print $2}' | submodule-and-sub-submodule-default-brancher
 git submodule status comm-libs \
  | awk -F ' ' '{print $2}' | submodule-and-sub-submodule-default-brancher
 git submodule status rocm-libraries \
  | awk -F ' ' '{print $2}' | submodule-and-sub-submodule-default-brancher
 git submodule status rocm-systems \
  | awk -F ' ' '{print $2}' \
   | xargs -I{} sh -c 'cd {} ; echo "all submodules are in rocprofiler-sdk and rocprofiler-register" ;\
    git submodule foreach --recursive \
     "git rev-parse --abbrev-ref origin/HEAD |  rg -v Enter | cut -c8- | xargs -n 1 git checkout && git pull "'
  pushd rocm-systems/projects/rocprofiler-sdk/external/perfetto ; git checkout eb5ef24 ; popd
  pushd rocm-systems/projects/rocprofiler-sdk/external/gotcha ; git checkout b944da1 ; popd
  pushd rocm-systems/projects/rocprofiler-sdk/external/fmt ; git checkout 0bffed8 ; popd
  pushd rocm-systems/projects/rocprofiler-sdk/external/cereal ; git checkout e736e75 ; popd
 echo 'reapply patches that checkout removed'
 pushd patches ;
  fd -t f 'Fix-find'  \
   | xargs -I{} sh -c 'cd ../base/amdsmi \
    ; patch -Nlp1 <"../../patches/"{}';
  fd -t f 'Find-rocm' \
   | xargs -I{} sh -c 'cd ../rocm-libraries \
    ;  patch -Nlp1 <"../patches/"{}';
  fd -t f 'enumerate' \
   | xargs -I{} sh -c 'cd ../rocm-libraries \
    ;  patch -Nlp1 <"../patches/"{}';
  popd ;
 echo 'next patch is not needed if triton/pytorch didnt block compiler checkout default branch'
 pushd ./compiler/amd-llvm/compiler-rt/lib/sanitizer_common/
  patch -Nlp1 << EOF 
diff -aur temptes/sanitizer_common_interceptors_ioctl.inc bbbbbbbb/sanitizer_common_interceptors_ioctl.inc
--- temptes/sanitizer_common_interceptors_ioctl.inc 2025-10-16 00:40:37.393039930 -0500
+++ bbbbbbbb/sanitizer_common_interceptors_ioctl.inc    2025-10-16 00:42:15.723673501 -0500
@@ -338,17 +338,9 @@
   _(SOUND_PCM_WRITE_CHANNELS, WRITE, sizeof(int));
   _(SOUND_PCM_WRITE_FILTER, WRITE, sizeof(int));
   _(TCFLSH, NONE, 0);
-#if SANITIZER_GLIBC
-  _(TCGETA, WRITE, struct_termio_sz);
-#endif
   _(TCGETS, WRITE, struct_termios_sz);
   _(TCSBRK, NONE, 0);
   _(TCSBRKP, NONE, 0);
-#if SANITIZER_GLIBC
-  _(TCSETA, READ, struct_termio_sz);
-  _(TCSETAF, READ, struct_termio_sz);
-  _(TCSETAW, READ, struct_termio_sz);
-#endif
   _(TCSETS, READ, struct_termios_sz);
   _(TCSETSF, READ, struct_termios_sz);
   _(TCSETSW, READ, struct_termios_sz);
diff -aur temptes/sanitizer_platform_limits_posix.cpp bbbbbbbb/sanitizer_platform_limits_posix.cpp
--- temptes/sanitizer_platform_limits_posix.cpp 2025-10-16 00:40:07.862171218 -0500
+++ bbbbbbbb/sanitizer_platform_limits_posix.cpp    2025-10-16 00:42:51.115725190 -0500
@@ -485,9 +485,6 @@
   unsigned struct_input_id_sz = sizeof(struct input_id);
   unsigned struct_mtpos_sz = sizeof(struct mtpos);
   unsigned struct_rtentry_sz = sizeof(struct rtentry);
-#if SANITIZER_GLIBC || SANITIZER_ANDROID
-  unsigned struct_termio_sz = sizeof(struct termio);
-#endif
   unsigned struct_vt_consize_sz = sizeof(struct vt_consize);
   unsigned struct_vt_sizes_sz = sizeof(struct vt_sizes);
   unsigned struct_vt_stat_sz = sizeof(struct vt_stat);
diff -aur temptes/sanitizer_platform_limits_posix.h bbbbbbbb/sanitizer_platform_limits_posix.h
--- temptes/sanitizer_platform_limits_posix.h   2025-10-16 00:40:19.094256617 -0500
+++ bbbbbbbb/sanitizer_platform_limits_posix.h  2025-10-16 00:43:00.723836368 -0500
@@ -1043,7 +1043,6 @@
 extern unsigned struct_input_absinfo_sz;
 extern unsigned struct_input_id_sz;
 extern unsigned struct_mtpos_sz;
-extern unsigned struct_termio_sz;
 extern unsigned struct_vt_consize_sz;
 extern unsigned struct_vt_sizes_sz;
 extern unsigned struct_vt_stat_sz;
EOF
 popd ;
 echo 'more than 8 will Out-Of-Memory my 64GB'
 sed -i 's/_JOBS ""/_JOBS "8"/' ./compiler/amd-llvm/llvm/cmake/modules/HandleLLVMOptions.cmake
 sed -i 's/# Version configuration/-DLLVM_PARALLEL_COMPILE_JOBS=8 -DLLVM_PARALLEL_LINK_JOBS=8/'  ./compiler/CMakeLists.txt;
 sed -i "$(echo $(grep -n "cxx_std_20" ./rocm-libraries/shared/rocroller/CMakeLists.txt  | cut -f1 -d:)"+1"|bc) i \
   target_include_directories(rocroller PUBLIC $<BUILD_INTERFACE:/usr/lib/gcc/x86_64-pc-linux-gnu/14.3.1/include/c++>)" \
   ./rocm-libraries/shared/rocroller/CMakeLists.txt
 sed -i 's/commit_hipify(args)/# commit_hipify(args)/' ./external-builds/pytorch/repo_management.py
 sed -i 's/AMDGPU;X86/AMDGPU;X86;NVPTX/' ./compiler/pre_hook_amd-llvm.cmake
 sed -i 's/AMDGPU_ARCH/AMDGPU_ARCH\nOFFLOAD_ARCH/' ./compiler/pre_hook_amd-llvm.cmake
 sed -i 's/gfx908 gfx90a gfx942 gfx950.*/gfx908 gfx90a gfx942 gfx950 gfx1103\)/' ./ml-libs/CMakeLists.txt
 sed -i 's/CMAKE_ARGS/CMAKE_ARGS\n      -DCMAKE_BUILD_PARALLEL_LEVEL=8/' ./ml-libs/CMakeLists.txt
 sed -i 's@gfx908;gfx90a;gfx942;gfx950;gfx10-3-generic;gfx11-generic;gfx12-generic@gfx11-generic;gfx1103@' ./ml-libs/composable_kernel/CMakeLists.txt
 sed -i 's@gfx908;gfx90a;gfx942;gfx950;gfx10-3-generic;gfx11-generic;gfx12-generic@gfx11-generic;gfx1103@' ./rocm-libraries/projects/composablekernel/CMakeLists.txt
```
# Make nspawn container. 
```
sudo pacman -Syu
sudo mkdir -p $CONTAINER && cd $CONTAINER/..
echo 'replacing fftw with fftw-amd works btw'
sudo pacstrap -K -c $CONTAINER  base base-devel nano foot-terminfo \
  uv gcc-fortran git git-lfs ninja pkgconf gvim patchelf automake \
  libglvnd python-pipenv ccache gcc14 gcc14-fortran fftw \
  devtools rust bc ripgrep bat less ;
 sudo systemd-nspawn -D $CONTAINER sh -c "
  echo 'omc' | passwd --stdin ;
  useradd -m user ;  gpasswd  -a user video ;  gpasswd  -a user render ;
  echo 'user' | passwd user --stdin
  echo '/home/user/TheRock/build/dist/rocm/lib' > /etc/ld.so.conf.d/rocm.conf ;
  sed -i 's/^# Defaults targetpw/ Defaults targetpw/' /etc/sudoers ;
  sed -i 's/^# ALL ALL=/ ALL ALL=/' /etc/sudoers ;
  echo 'use hosts pkg instead of downloading everything again' ;
   sed -i 's/^#CacheDir.*/CacheDir    = \/var\/cache\/pacman\/pkg\/\nCacheDir    = \/host\/pacman\/pkg\//' /etc/pacman.conf  ;
  sudo -u user mkdir -p /home/user/TheRock/ ;
  sudo -u user mkdir -p /home/user/.triton/ ;
  sudo -u user mkdir -p /home/user/.cache/uv/ ;
  sudo -u user mkdir -p /home/user/.cache/ccache/ ;
  sudo -u user mkdir -p /home/user/pkgbuild ;
  pushd /home/user/pkgbuild ;
   sed -i 's/purge debug lto/purge !debug lto/' /etc/makepkg.conf ;
   sed -i 's/x86-64/native/' /etc/makepkg.conf
   sed -i 's/generic/native/' /etc/makepkg.conf
   curl 'https://aur.archlinux.org/cgit/aur.git/plain/PKGBUILD?h=aocl-utils' -o PKGBUILD-aocl-utils
   curl 'https://aur.archlinux.org/cgit/aur.git/plain/PKGBUILD?h=aocl-blis' -o PKGBUILD-aocl-blis
   curl 'https://aur.archlinux.org/cgit/aur.git/plain/PKGBUILD?h=aocl-libflame' -o PKGBUILD-aocl-libflame
   sed -i '/source=/d' PKGBUILD-aocl-libflame PKGBUILD-aocl-blis
   sed -i 's@prepare() {@prepare() {\n git clone --depth 1  https://github.com/amd/libflame@' PKGBUILD-aocl-libflame
   sed -i 's@prepare() {@prepare() {\n git clone --depth 1 -b dev https://github.com/amd/blis@' PKGBUILD-aocl-blis
   sed -i 's@srcdir/blis.*@srcdir/blis@' PKGBUILD-aocl-blis
   sed -i 's@srcdir/libflame-\$pkgver@srcdir/libflame@' PKGBUILD-aocl-libflame
   sed -i '/export CFLAGS/d' PKGBUILD-aocl-libflame PKGBUILD-aocl-blis
   sed -i '/export CXXFLAGS/d' PKGBUILD-aocl-libflame PKGBUILD-aocl-blis
   sed -i '/gcc15.patch/d' PKGBUILD-aocl-blis
   sed -i 's@pkgconfig/blas.pc@pkgconfig/blas.pc\nln -s /usr/lib/pkgconfig/blis-mt.pc \$pkgdir/usr/lib/pkgconfig/blis.pc@' PKGBUILD-aocl-blis
   sed -i 's@pkgconfig/blis@pkgconfig/blis-mt@' PKGBUILD-aocl-blis
   sed -i 's@lib/lapack@lib/pkgconfig/lapack@' PKGBUILD-aocl-libflame
   sudo -u user makepkg --printsrcinfo -p PKGBUILD-aocl-utils | rg makedepends | cut -d' ' -f3 | xargs pacman -S  --noconfirm
   sudo -u user makepkg --printsrcinfo -p PKGBUILD-aocl-blis | rg makedepends | cut -d' ' -f3 | xargs pacman -S  --noconfirm
   sudo -u user makepkg --printsrcinfo -p PKGBUILD-aocl-libflame | rg makedepends | cut -d' ' -f3 | xargs pacman -S  --noconfirm
   sudo -u user makepkg --noconfirm -C --skipchecksums -p PKGBUILD-aocl-utils
    echo 'omc' | sudo -S pacman -U aocl-utils*zst --noconfirm
   sudo -u user makepkg --noconfirm -C --skipchecksums -p PKGBUILD-aocl-blis
    echo 'omc' | sudo -S pacman -U aocl-blis*zst --noconfirm
   sudo -u user makepkg --noconfirm -C --skipchecksums -p PKGBUILD-aocl-libflame
    echo 'omc' | sudo -S pacman -U aocl-libflame*zst --noconfirm
   popd ;
  exit;"
```
nspawn? bind mounts host folder onto container folder & last 4 lines allow container to use gpu and NPU, not that ive found something that uses pheonix npu
```
 echo 'WOO nspawn!' 
 systemd-nspawn -D $CONTAINER \
    --bind=$HOST/TheRock/:/home/user/TheRock/ \
    --bind=$HOST/.triton/:/home/user/.triton/ \
    --bind=$HOST/.cache/uv/:/home/user/.cache/uv/ \
    --bind=$HOST/.cache/ccache/:/home/user/.cache/ccache/ \
    --property="DeviceAllow=char-drm rw" \
    --property="DeviceAllow=char-dma_heap rw" \
    --property="DeviceAllow=char-accel rw" \
    --bind=/dev/dri --bind=/dev/kfd --bind=/dev/shm --bind=/dev/accel
```
# building in the container
```
systemd-nspawn -D $CONTAINER \
    --bind=$HOST/ai/TheRock/:/home/user/TheRock/ \
    --bind=$HOST/.triton/:/home/user/.triton/ \
    --bind=$HOST/.cache/uv/:/home/user/.cache/uv/ \
    --bind=$HOST/.cache/ccache/:/home/user/.cache/ccache/ \
    --bind=/var/cache/pacman/pkg/:/host/pacman/pkg/ \
    --property="DeviceAllow=char-drm rw" \
    --property="DeviceAllow=char-dma_heap rw" \
    --property="DeviceAllow=char-accel rw" \
    --bind=/dev/dri --bind=/dev/kfd --bind=/dev/shm --bind=/dev/accel sh -c "
 su -l user -c \"
  cd /home/user/TheRock ;
  python3 -m venv venv;
  source venv/bin/activate ;
  uv pip install 'cmake<4' ;
  uv pip install -r requirements.txt ;
  export PATH=$PATH:/home/user/TheRock/build/dist/rocm/bin ;
  export HIP_DEVICE_LIB_PATH=/home/user/TheRock/build/compiler/amd-llvm/dist/lib/llvm/amdgcn/bitcode/ ;
  cmake -B build -GNinja \
   -DCMAKE_C_COMPILER_LAUNCHER=ccache \
   -DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
   -DTHEROCK_AMDGPU_TARGETS='gfx1103' ;
  cmake --build build --target therock-dist ;
  mkdir /home/user/python-THEROCK ;
  ./build_tools/build_python_packages.py \
    --artifact-dir ./build/artifacts \
    --dest-dir /home/user/python-THEROCK ;
  uv pip install /home/user/python-THEROCK/dist/rocm* ;
  rocm-sdk test ;
  export PYTORCH_ROCM_ARCH=gfx1103 ; export USE_FLASH_ATTENTION=1 ; export USE_MEM_EFF_ATTENTION=1 ; export TORCH_ROCM_AOTRITON_ENABLE_EXPERIMENTAL=1 ;
  pushd ./external-builds/pytorch ;
  pushd pytorch ; git reset --hard ; git clean -f ; popd;
  python pytorch_torch_repo.py checkout ;
   sed -z -i 's@blis@blis-mt@3'              ./pytorch/cmake/Modules/FindBLAS.cmake
   sed -z -i 's@blis@blis-mt@7'              ./pytorch/cmake/Modules/FindBLAS.cmake
   sed -z -i 's@blis@FLAME@4'    ./pytorch/cmake/Modules/FindBLAS.cmake
    sed -i 's@/usr/include/blis@/usr/include@'       ./pytorch/cmake/Modules/FindBLIS.cmake
    sed -i 's@/usr/lib/blis@/usr/lib@'           ./pytorch/cmake/Modules/FindBLIS.cmake
    sed -i 's@NAMES blis PATHS@NAMES blis-mt PATHS@'     ./pytorch/cmake/Modules/FindBLIS.cmake
  python pytorch_audio_repo.py checkout
  python pytorch_vision_repo.py checkout
  pushd triton ;
   git reset --hard ; git clean -f ;
   patch -Nl <  ../patches/triton/nightly/triton/base/0001-Keep-only-one-plus-character-in-version-suffix.patch
   popd
  python pytorch_triton_repo.py checkout
  mkdir /home/user/python-PYTORCH ;
   python build_prod_wheels.py build \
    --output-dir /home/user/python-PYTORCH  2>stop-spaming.log  1>my-terminal.log  ;
   popd ;
  exit;\"
 exit;"
```

# Done with the container
```
 cd $HOST/ai/sdnext ;
 python3 -m venv venv && source venv/bin/activate ;
  uv pip install $CONTAINER/home/user/python-THEROCK/dist/rocm_sdk_[cl]*
  uv pip install $CONTAINER/home/user/python-THEROCK/dist/rocm-0.1.dev0.tar.gz
  uv pip install $CONTAINER/home/user/python-PYTORCH/*
 pushd ../flash-attention
  uv pip install packaging
  FLASH_ATTENTION_TRITON_AMD_AUTOTUNE="TRUE"  FLASH_ATTENTION_TRITON_AMD_ENABLE="TRUE" uv pip install . --no-build-isolation
  popd
 sed -i 's@.*nightly/rocm6.4.*@# SKIP@' installer.py
 sed -i 's@raise NotImplementedError@path = os.path.join(_rocm_sdk_core.__path__[0], "lib", "libamdhip64.so.7")@' modules/rocm.py
 export HSA_OVERRIDE_GFX_VERSION=11.0.3
 export PYTORCH_ROCM_ARCH=gfx1103
 export FLASH_ATTENTION_TRITON_AMD_ENABLE=TRUE
 export ROCM_PATH=$HOST/ai/sdnext/venv/lib/python3.13/site-packages/_rocm_sdk_core/
 export HIP_DEVICE_LIB_PATH=$HOST/ai/sdnext/venv/lib/python3.13/site-packages/_rocm_sdk_core/lib/llvm/amdgcn/bitcode/
 export PATH=$PATH:$HOST/ai/sdnext/venv/lib/python3.13/site-packages/_rocm_sdk_core/bin
 echo 'gfx1103'
  export MIOPEN_DEBUG_AMD_WINOGRAD_FURY_RXS_F2X3=0
  export MIOPEN_DEBUG_CONV_DIRECT_NAIVE_CONV_FWD=0
 ./webui.sh --use-rocm --experimental --use-nightly --uv
```
# useful environment variables
```
#export TORCHINDUCTOR_COMPILE_THREADS=1 ;echo 'restrain miopen? from maxing cpu'
#export LD_PRELOAD=libmimalloc.so.2 ;echo 'i never noticed a difference'
#export LD_PRELOAD=libjemalloc.so.2  

export PYTORCH_TUNABLEOP_ENABLED=1
export TORCH_ROCM_AOTRITON_ENABLE_EXPERIMENTAL=1
export FLASH_ATTENTION_TRITON_AMD_ENABLE=TRUE 
export PYTORCH_ROCM_ARCH="gfx1103"
export HSA_OVERRIDE_GFX_VERSION=11.0.3

export MIOPEN_FIND_MODE=FAST
#export MIOPEN_FIND_ENFORCE=SEARCH
export MIOPEN_DEBUG_AMD_WINOGRAD_FURY_RXS_F2X3=0
export MIOPEN_DEBUG_CONV_DIRECT_NAIVE_CONV_FWD=0
echo 'block above 2 on gfx1103'

#export AMD_LOG_LEVEL=3
#export AMD_LOG_LEVEL_FILE="miopen-shadername-spam"
#export AMD_LOG_MASK=80000
echo 'tried comfy for the first time, and same venv and exports worked.'
```
# Why so much filesize?
 ```
rm -ri $CONTAINER/home/user/python-THEROCK/*
rm -ri $CONTAINER/home/user/python-PYTORCH/*

echo 'below is with VENV active';
 du -t+222M -h -d1 $(uv cache dir)"/archive-v0/" | head -n-1 | awk -F ' ' '{print $2}' | xargs rm -r

 echo "behold horror! if inside uv cache, there be installed items larger than remaining space left on partition, where uv directory is located, remove said items"
  du -t+$(df -h  $(uv cache dir) | awk -F ' ' '{print $4}' | tail -n1) -h -d0 $(uv cache dir)"/archive-v0/"* | awk -F ' ' '{print $2}' | xargs rm -r

 uv pip uninstall rocm rocm-sdk-core rocm-sdk-devel rocm-sdk-libraries-gfx1103
 uv pip uninstall pytorch-triton-rocm torch torchvision torchaudio flash-attn
#uv cache clean
```
