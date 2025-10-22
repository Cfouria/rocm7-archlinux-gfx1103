# rocm7-archlinux-gfx1103
Just notes on how I built rocm-gfx1103.  
Encountered systemd-nspawn and restarted my attempts at ROCM. 

#  total    ~230GB LOTS of disk space usage  
THEROCK     ~120GB  ( git clone --depth 1 --revision ${_commit} ... bless)  
uv cache    ~30GB  
container   ~35GB   (when building wheel packages from THEROCK, 50GB is reached and ~15GB is 'cleaned up')  
/var        ~3GB  
/usr        ~5GB    (if container is an indicator)  
ccache      ~5GB  
comgr       ~5GB    (code object manager ~MIopen)  
sdnext      ~15GB  


# This is as I did it, with some re-arrangement.
```
USER=c4
HOST=/home/$USER
CONTAINER=/opt/myContainer13

mkdir -p $HOST && cd $HOST
git clone --depth 1 -b dev https://github.com/vladmandic/sdnext
git clone --depth 1 -b main_perf https://github.com/ROCmSoftwarePlatform/flash-attention
git clone --depth 1 --shallow-submodules https://github.com/ROCm/TheRock
cd ./TheRock
 pushd ./rocm-systems ;
 echo 'THEROCK successfully builds when changing everything to default...'
 echo 'but pytorch triton seems to do something "special" with llvm...'
 echo 'which requires ./compiler/amd-llvm to be specific sigh...'
 echo 'fails when launching sdnext, so way after many hours of compiling' ;
  git submodule foreach --recursive \
   pwd | rg -v Enter | \
    xargs -I{} sh -c \
     'cd {} ; git checkout $(basename $(git rev-parse --abbrev-ref origin/HEAD)) ; git pull'
  echo "these 4 fail if you try to keep on newest branch"
   pushd ./projects/rocprofiler-sdk/external/cereal  ; git checkout e736e75 ; popd
   pushd ./projects/rocprofiler-sdk/external/fmt     ; git checkout 0bffed8 ; popd
   pushd ./projects/rocprofiler-sdk/external/gotcha  ; git checkout b944da1 ; popd
   pushd ./projects/rocprofiler-sdk/external/perfetto    ; git checkout eb5ef24 ; popd
  popd
 python ./build_tools/fetch_sources.py
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
EOF ; popd ; echo 'likely not need this patch in a few weeks when TheRock compiler is updated'
 echo 'OOM without next 2'
  sed -i 's/_JOBS ""/_JOBS "8"/' ./compiler/amd-llvm/llvm/cmake/modules/HandleLLVMOptions.cmake
  sed -i 's/# Version configuration/-DLLVM_PARALLEL_COMPILE_JOBS=8 -DLLVM_PARALLEL_LINK_JOBS=8/'  ./compiler/CMakeLists.txt;
 echo 'rocRoller and stl_algobase.h (gcc14, while archlinux is on gcc15)
  sed -i "$(echo $(grep -n "cxx_std_20" ./rocm-libraries/shared/rocroller/CMakeLists.txt  | cut -f1 -d:)"+1"|bc) i \
   target_include_directories(rocroller PUBLIC $<BUILD_INTERFACE:/usr/lib/gcc/x86_64-pc-linux-gnu/14.3.1/include/c++>)" \
   ./rocm-libraries/shared/rocroller/CMakeLists.txt
 echo "the next 4 all error with 'hip/hip_runtime.h' file not found"
  sed -i "$(echo $(grep -n "executable.roctx_test" ./rocm-systems/projects/roctracer/test/CMakeLists.txt  | cut -f1 -d:)"+1"|bc) i \
   target_include_directories(roctx_test PRIVATE "/home/user/TheRock/build/core/clr/dist/include")" \
   ./rocm-systems/projects/roctracer/test/CMakeLists.txt
  sed -i 's/O3/O3 -I\/home\/user\/TheRock\/build\/math-libs\/BLAS\/hipBLAS-common\/dist\/include/' \
   ./rocm-libraries/projects/hipblaslt/device-library/matrix-transform/CMakeLists.txt  ;
  sed -i 's/c++17",/c++17", "-I\/home\/user\/TheRock\/build\/math-libs\/BLAS\/hipBLAS-common\/dist\/include",/' \
   ./rocm-libraries/projects/hipblaslt/tensilelite/Tensile/Toolchain/Component.py ;
  sed -i 's/"-I", outputPath/"-I", outputPath, "-I\/home\/user\/TheRock\/build\/dist\/rocm\/include"/' \
   ./rocm-libraries/shared/tensile/Tensile/BuildCommands/SourceCommands.py ;
 echo 'no i do not want to make a commit'
  sed -i 's/commit_hipify(args)/# commit_hipify(args)/' ./external-builds/pytorch/repo_management.py
 echo 'rocm-sdk test fails, and triton fails'
  sed -i 's/AMDGPU;X86/AMDGPU;X86;NVPTX/' ./compiler/pre_hook_amd-llvm.cmake
 echo 'cant find amdgpu-arch? llvm changed it to offload_arch'
  sed -i 's/AMDGPU_ARCH/AMDGPU_ARCH\nOFFLOAD_ARCH/' ./compiler/pre_hook_amd-llvm.cmake
 echo 'gfx1103 not supported in CK? yolo, hey it works'
  sed -i 's/gfx908 gfx90a gfx942 gfx950.*/gfx908 gfx90a gfx942 gfx950 gfx1103\)/' ./ml-libs/CMakeLists.txt
  sed -i 's/CMAKE_ARGS/CMAKE_ARGS\n      -DCMAKE_BUILD_PARALLEL_LEVEL=8/' ./ml-libs/CMakeLists.txt
  sed -i 's@gfx908;gfx90a;gfx942;gfx950;gfx10-3-generic;gfx11-generic;gfx12-generic@gfx11-generic;gfx1103@' \
   ./ml-libs/composable_kernel/CMakeLists.txt
  sed -i 's@gfx908;gfx90a;gfx942;gfx950;gfx10-3-generic;gfx11-generic;gfx12-generic@gfx11-generic;gfx1103@' \
   ./rocm-libraries/projects/composablekernel/CMakeLists.txt
```
# Make nspawn container. Take care your in container when changing passwd.
```
sudo mkdir -p $CONTAINER && cd $CONTAINER/..
echo 'i dont remember why i used su and not sudo'
su
 echo 'a few are for my convience (nano foot devtools? rust bc ripgrep bat less)'
 pacstrap -K -c $CONTAINER  base base-devel nano foot-terminfo \
  uv gcc-fortran git git-lfs ninja pkgconf gvim patchelf automake \
  libglvnd python-pipenv ccache gcc14 gcc14-fortran fftw \
  devtools rust bc ripgrep bat less ;
 systemd-nspawn -D $CONTAINER ;
  passwd
  useradd -m user
  gpasswd  -a user video
  gpasswd  -a user render
  passwd user
  echo '/home/user/TheRock/build/dist/rocm/lib' > /etc/ld.so.conf.d/rocm.conf
  sed -i 's/^# Defaults targetpw/ Defaults targetpw/' /etc/sudoers 
  sed -i 's/^# ALL ALL=/ ALL ALL=/' /etc/sudoers 
  echo 'use hosts pkg instead of downloading everything again (edited to not leak my filepaths)'
   sed -i 's/^#CacheDir.*/CacheDir    = \/var\/cache\/pacman\/pkg\//' /etc/pacman.conf 
  mkdir -p /home/user/TheRock/
   chown user:user /home/user/TheRock/
  mkdir -p /home/user/.triton/
   chown user:user /home/user/.triton
  mkdir -p /home/user/.cache/uv/
   chown user:user /home/user/.cache
   chown user:user /home/user/.cache/uv/
  mkdir -p /home/user/.cache/ccache/
   chown user:user /home/user/.cache/ccache/
  echo 'could maybe use blas or intel mkl from archlinux repo'
  echo 'I think the blas missing errors were during pytorch build'
  mkdir -p /home/user/pkgbuild;
   pushd /home/user/pkgbuild ;
    sed -i 's/x86-64/native/' /etc/makepkg.conf
    sed -i 's/generic/native/' /etc/makepkg.conf
    curl 'https://aur.archlinux.org/cgit/aur.git/plain/PKGBUILD?h=aocl-utils' -o PKGBUILD-aocl-utils
    curl 'https://aur.archlinux.org/cgit/aur.git/plain/PKGBUILD?h=aocl-blis' -o PKGBUILD-aocl-blis
    curl 'https://aur.archlinux.org/cgit/aur.git/plain/PKGBUILD?h=aocl-libflame' -o PKGBUILD-aocl-libflame
    sed -i 's@source=@# source=@' PKGBUILD-aocl-libflame PKGBUILD-aocl-blis 
    sed -i 's@prepare() {@prepare() {\n git clone --depth 1  https://github.com/amd/libflame@' PKGBUILD-aocl-libflame
    sed -i 's@prepare() {@prepare() {\n git clone --depth 1 -b dev https://github.com/amd/blis@' PKGBUILD-aocl-blis
    sed -i 's@$srcdir/blis-$_tag_str@$srcdir/blis@' PKGBUILD-aocl-blis 
    sed -i 's@$srcdir/libflame-$pkgver@$srcdir/libflame@' PKGBUILD-aocl-libflame 
    sed -i 's@export CFLAGS@# export CFLAGS@' PKGBUILD-aocl-libflame PKGBUILD-aocl-blis
    sed -i 's@export CXXFLAGS@# export CXXFLAGS@' PKGBUILD-aocl-libflame PKGBUILD-aocl-blis
    sed -i 's@"gcc15.patch@# "gcc15.patch@' PKGBUILD-aocl-blis
    sed -i 's@patch -p1@# patch -p1@' PKGBUILD-aocl-blis
    sed -i 's@pkgconfig/blas.pc@pkgconfig/blas.pc\nln -s /usr/lib/pkgconfig/blis-mt.pc $pkgdir/usr/lib/pkgconfig/blis.pc@' PKGBUILD-aocl-blis
    sed -i 's@pkgconfig/blis@pkgconfig/blis-mt@' PKGBUILD-aocl-blis
    sed -i 's@lib/lapack@lib/pkgconfig/lapack@' PKGBUILD-aocl-libflame 
    makepkg -Csi --skipchecksums -p PKGBUILD-aocl-utils
    makepkg -Csi --skipchecksums -p PKGBUILD-aocl-blis
    makepkg -Csi --skipchecksums -p PKGBUILD-aocl-libflame
    popd ; 
  exit
 echo 'WOO nspawn! first 4 mean dont redownload when blowing up the container'
 echo 'last 4 are to allow gpu & !NPU! not that ive found something that uses pheonix npu...'
 systemd-nspawn -b -D $CONTAINER \
    --bind=$HOST/TheRock/:/home/user/TheRock/ \
    --bind=$HOST/.triton/:/home/user/.triton/ \
    --bind=$HOST/.cache/uv/:/home/user/.cache/uv/ \
    --bind=$HOST/.cache/ccache/:/home/user/.cache/ccache/ \
    --property="DeviceAllow=char-drm rw" \
    --property="DeviceAllow=char-dma_heap rw" \
    --property="DeviceAllow=char-accel rw" \
    --bind=/dev/dri --bind=/dev/kfd --bind=/dev/shm --bind=/dev/accel
```
# login to the container
```
 echo 'login as user in container'
  cd TheRock
   python3 -m venv venv && source venv/bin/activate ;
   uv pip install 'cmake<4' ;
   uv pip install -r requirements.txt ;  
   export PATH=$PATH:/home/user/TheRock/build/dist/rocm/bin ;
   cmake -B build -GNinja \
    -DCMAKE_C_COMPILER_LAUNCHER=ccache \
    -DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
    -DTHEROCK_AMDGPU_TARGETS='gfx1103';
   cmake --build build --target therock-dist ;
   mkdir /home/user/python-THEROCK ;
    ./build_tools/build_python_packages.py \
     --artifact-dir ./build/artifacts \
     --dest-dir /home/user/python-THEROCK
   uv pip install /home/user/python-THEROCK/dist/rocm*
   rocm-sdk test
```

# Exit the container Ctrl+] Ctrl+] p or ```machinectl poweroff container``` &&& exit su !!!
```
 cd $HOST/sdnext ;
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
 export ROCM_PATH=$HOST/TheRock/build/dist/rocm/ 
 export HIP_DEVICE_LIB_PATH=$HOST/TheRock/build/dist/rocm/lib/llvm/amdgcn/bitcode/ 
 export PATH="$PATH:$HOST/TheRock/build/dist/rocm/bin"

echo 'lazy kicks in'
export TORCHINDUCTOR_COMPILE_THREADS=1 ;echo 'restrain miopen? from maxing cpu'
#export LD_PRELOAD=libmimalloc.so.2 ;echo 'i never noticed a difference'
#export LD_PRELOAD=libjemalloc.so.2  
#export MALLOC_CONF=oversize_threshold:1,background_thread:true,metadata_thp:auto,dirty_decay_ms:9000000000,muzzy_decay_ms:9000000000 ;echo '???'

export PYTORCH_TUNABLEOP_ENABLED=1
export TORCH_ROCM_AOTRITON_ENABLE_EXPERIMENTAL=1
export FLASH_ATTENTION_TRITON_AMD_ENABLE=TRUE 
export PYTORCH_ROCM_ARCH="gfx1103"
export HSA_OVERRIDE_GFX_VERSION=11.0.3

export MIOPEN_FIND_MODE=FAST
#export MIOPEN_FIND_ENFORCE=SEARCH
export MIOPEN_DEBUG_AMD_WINOGRAD_FURY_RXS_F2X3=0 ;echo 'yes gfx1103 still complains about both these after 3months?'
export MIOPEN_DEBUG_CONV_DIRECT_NAIVE_CONV_FWD=0

#export AMD_LOG_LEVEL=3
#export AMD_LOG_LEVEL_FILE="third16"
#export AMD_LOG_MASK=80000
 ./webui.sh --use-rocm --experimental --use-nightly --uv
echo 'tried comfy for the first time, and same venv and exports worked.'
```
# Why so much filesize?
 ```
rm -ri $CONTAINER/home/user/python-THEROCK/*

echo 'below is with VENV active';
 du -t+222M -h -d1 $(uv cache dir)"/archive-v0/" | head -n-1 | awk -F ' ' '{print $2}' | xargs rm -r

 echo "behold horror! if inside uv cache, there be installed items larger than remaining space left on partition, where uv directory is located, remove said items"
  du -t+$(df -h  $(uv cache dir) | awk -F ' ' '{print $4}' | tail -n1) -h -d0 $(uv cache dir)"/archive-v0/"* | awk -F ' ' '{print $2}' | xargs rm -r

 uv pip uninstall rocm rocm-sdk-core rocm-sdk-devel rocm-sdk-libraries-gfx1103
 #uv cache clean
```
