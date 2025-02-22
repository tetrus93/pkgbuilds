# Maintainer: samicrusader <hi@samicrusader.me>
# Contributor: Sven-Hendrik Haase <svenstaro@archlinux.org>
# Contributor: Thomas Baechler <thomas@archlinux.org>
# Contributor: James Rayner <iphitus@gmail.com>

pkgbase=nvidia-snowman-utils
pkgname=('nvidia-snowman-utils' 'opencl-nvidia-snowman' 'nvidia-snowman-dkms')
pkgver=535.54.03
_hostver=525.105.14
pkgrel=3
arch=('x86_64')
url='https://discord.gg/qh6dvPtxvb'
license=('custom')
options=('!strip')
makedepends=('git' 'unzip' 'perl' 'cargo' 'rust' 'sed' 'libarchive' 'patchelf')
_pkg='NVIDIA-GRID-Linux-KVM-525.105.14-525.105.17-528.89'
_consumerdriver="NVIDIA-Linux-x86_64-$pkgver"
_vgpukvmdriver="NVIDIA-Linux-x86_64-$_hostver-vgpu-kvm"
_griddriver="NVIDIA-Linux-x86_64-525.105.17-grid"
_mergeddriver="$_consumerdriver-merged-vgpu-kvm-patched"
source=('nvidia-drm-outputclass.conf'
        'nvidia-utils.sysusers'
        'nvidia.rules'
        'nvidia-grid.conf'
        "https://us.download.nvidia.com/XFree86/Linux-x86_64/${pkgver}/${_consumerdriver}.run"
        'NVIDIA-GRID-Linux-KVM-525.105.14-525.105.17-528.89.zip'
        'git+https://github.com/VGPU-Community-Drivers/vGPU-Unlock-patcher.git#branch=experimental/535.54'
        'git+https://github.com/DualCoder/vgpu_unlock.git#commit=f432ffc8b7ed245df8858e9b38000d3b8f0352f4'
        'git+https://github.com/mbilker/vgpu_unlock-rs#commit=44d5bb32ecd8bdcfe374772c31078a6e4eef921f')
sha256sums=('be99ff3def641bb900c2486cce96530394c5dc60548fc4642f19d3a4c784134d'
            'd8d1caa5d72c71c6430c2a0d9ce1a674787e9272ccce28b9d5898ca24e60a167'
            '4fbfd461f939f18786e79f8dba5fdb48be9f00f2ff4b1bb2f184dbce42dd6fc3'
            '7d243854807a3b1150285d579979a9f380b3955f5f77ab9f400287fd4fd36f81'
            '454764f57ea1b9e19166a370f78be10e71f0626438fb197f726dc3caf05b4082'
            'c8e12c15b881df35e618bdee1f141cbfcc7e112358f0139ceaa95b48e20761e0'
            'SKIP'
            'SKIP'
            'SKIP')
noextract=("${_pkg}.zip")

create_links() {
    # create soname links
    find "$pkgdir" -type f -name '*.so*' ! -path '*xorg/*' -print0 | while read -d $'\0' _lib; do
        _soname=$(dirname "${_lib}")/$(readelf -d "${_lib}" | grep -Po 'SONAME.*: \[\K[^]]*' || true)
        _base=$(echo ${_soname} | sed -r 's/(.*)\.so.*/\1.so/')
        [[ -e "${_soname}" ]] || ln -s $(basename "${_lib}") "${_soname}"
        [[ -e "${_base}" ]] || ln -s $(basename "${_soname}") "${_base}"
    done
}

prepare() {
    echo 'Fixing Git submodules...'
    cd "$srcdir/vGPU-Unlock-patcher/"
    git submodule init
    git config submodule.unlock.url "$srcdir/vgpu_unlock"
    git -c protocol.file.allow=always submodule update

    echo "Extracting $_pkg.zip..."
    cd "$srcdir"
    unzip "$srcdir/$_pkg.zip"
    mv "$srcdir/Guest_Drivers/$_griddriver.run" "$srcdir"
    mv "$srcdir/Host_Drivers/$_vgpukvmdriver.run" "$srcdir"
    rm -rf "$srcdir"/{*.pdf,Signing_Keys,Host_Drivers,Guest_Drivers}
    mv "$srcdir"/*.run "$srcdir/vGPU-Unlock-patcher/"

    echo 'Building merged driver...'
    cd "$srcdir/vGPU-Unlock-patcher"
    chmod +x "$srcdir/vGPU-Unlock-patcher/patch.sh"
    ./patch.sh --lk6-patches general-merge

    echo 'Building vgpu_unlock-rs...'
    cd "$srcdir/vgpu_unlock-rs"
    cargo build --release
    mv "$srcdir/vgpu_unlock-rs/target/release/libvgpu_unlock_rs.so" "$srcdir"
    sed -i '/^Type=forking/a Environment=LD_PRELOAD=/usr/lib/nvidia/libvgpu_unlock_rs.so' "$srcdir/vGPU-Unlock-patcher/$_mergeddriver/init-scripts/systemd"/nvidia-vgpu{d,-mgr}.service

    # https://github.com/keylase/nvidia-patch
    echo 'Removing Video Codec SDK concurrent encode limit...'
    sed -i 's/\xe8\xa5\x9e\xfe\xff\x85\xc0\x41\x89\xc4/\xe8\xa5\x9e\xfe\xff\x29\xc0\x41\x89\xc4/g' "$srcdir/vGPU-Unlock-patcher/$_mergeddriver/libnvidia-encode.so.$pkgver"

    echo 'Enabling NvFBC...'
    sed -i 's/\x83\xfe\x01\x73\x08\x48/\x83\xfe\x00\x72\x08\x48/' "$srcdir/vGPU-Unlock-patcher/$_mergeddriver/libnvidia-fbc.so.$pkgver"

    echo 'Preparing driver...'
    cd "$srcdir/vGPU-Unlock-patcher/$_mergeddriver"
    bsdtar -xf nvidia-persistenced-init.tar.bz2

    # return process list to nvidia-smi
    # original from https://gist.github.com/KonradCzerw/80d3b907946b5c940f50da7795364bc1
    # perl -pi -e 's/\x6E\x01\x00\x00\x74\x5A\x41\xC7/\x6E\x01\x00\x00\xEB\x5A\x41\xC7/g' "$srcdir/vGPU-Unlock-patcher/$_mergeddriver/libnvidia-ml.so.$pkgver"
    
    sed -i "s/__VERSION_STRING/${pkgver}/" "$srcdir/vGPU-Unlock-patcher/$_mergeddriver/kernel/dkms.conf"
    sed -i 's/__JOBS/`nproc`/' "$srcdir/vGPU-Unlock-patcher/$_mergeddriver/kernel/dkms.conf"
    sed -i 's/__DKMS_MODULES//' "$srcdir/vGPU-Unlock-patcher/$_mergeddriver/kernel/dkms.conf"
    sed -i '$iBUILT_MODULE_NAME[0]="nvidia"\
DEST_MODULE_LOCATION[0]="/kernel/drivers/video"\
BUILT_MODULE_NAME[1]="nvidia-uvm"\
DEST_MODULE_LOCATION[1]="/kernel/drivers/video"\
BUILT_MODULE_NAME[2]="nvidia-modeset"\
DEST_MODULE_LOCATION[2]="/kernel/drivers/video"\
BUILT_MODULE_NAME[3]="nvidia-drm"\
DEST_MODULE_LOCATION[3]="/kernel/drivers/video"\
BUILT_MODULE_NAME[4]="nvidia-peermem"\
DEST_MODULE_LOCATION[4]="/kernel/drivers/video"\
BUILT_MODULE_NAME[5]="nvidia-vgpu-vfio"\
DEST_MODULE_LOCATION[5]="/kernel/drivers/video"' "$srcdir/vGPU-Unlock-patcher/$_mergeddriver/kernel/dkms.conf"

    # Gift for linux-rt guys
    sed -i 's/NV_EXCLUDE_BUILD_MODULES/IGNORE_PREEMPT_RT_PRESENCE=1 NV_EXCLUDE_BUILD_MODULES/' "$srcdir/vGPU-Unlock-patcher/$_mergeddriver/kernel/dkms.conf"
}

package_opencl-nvidia-snowman() {
    pkgdesc="OpenCL implemention for NVIDIA (snowman merge)"
    depends=('zlib')
    optdepends=('opencl-headers: headers necessary for OpenCL development')
    provides=('opencl-driver' 'opencl-nvidia')
    cd "$srcdir/vGPU-Unlock-patcher/$_mergeddriver"

    # OpenCL
    install -Dm644 nvidia.icd "${pkgdir}/etc/OpenCL/vendors/nvidia.icd"
    install -Dm755 "libnvidia-opencl.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-opencl.so.${pkgver}"

    create_links

    mkdir -p "${pkgdir}/usr/share/licenses"
    ln -s nvidia-snowman-utils "${pkgdir}/usr/share/licenses/opencl-nvidia" # ??
}

package_nvidia-snowman-dkms() {
    pkgdesc="NVIDIA drivers - module sources (snowman merge)"
    depends=('dkms' "nvidia-snowman-utils=$pkgver" 'libglvnd')
    provides=('NVIDIA-MODULE' 'nvidia' 'nvidia-dkms')
    conflicts=('NVIDIA-MODULE' 'nvidia')

    install -dm 755 "${pkgdir}"/usr/src
    cp -dr --no-preserve='ownership' "$srcdir/vGPU-Unlock-patcher/$_mergeddriver/kernel" "${pkgdir}/usr/src/nvidia-${pkgver}"

    install -Dt "${pkgdir}/usr/share/licenses/${pkgname}" -m644 "$srcdir/vGPU-Unlock-patcher/$_mergeddriver/LICENSE"
}

package_nvidia-snowman-utils() {
    pkgdesc="NVIDIA drivers utilities (snowman merge)"
    depends=('xorg-server' 'libglvnd' 'egl-wayland')
    optdepends=('nvidia-settings: configuration tool'
                'xorg-server-devel: nvidia-xconfig'
                'opencl-nvidia: OpenCL support'
                'mdevctl: mediated device control'
                'libvirt: use mediated devices in qemu/kvm easily')
    conflicts=('nvidia-libgl')
    provides=('vulkan-driver' 'opengl-driver' 'nvidia-libgl' 'nvidia-utils')
    replaces=('nvidia-libgl')
    install="${pkgname}.install"

    cd "$srcdir/vGPU-Unlock-patcher/$_mergeddriver"

    # Check http://us.download.nvidia.com/XFree86/Linux-x86_64/${pkgver}/README/installedcomponents.html
    # for hints on what needs to be installed where.

    # X driver
    install -Dm755 nvidia_drv.so "${pkgdir}/usr/lib/xorg/modules/drivers/nvidia_drv.so"

    # Wayland/GBM
    install -Dm755 libnvidia-egl-gbm.so.1* -t "${pkgdir}/usr/lib/"
    install -Dm644 15_nvidia_gbm.json "${pkgdir}/usr/share/egl/egl_external_platform.d/15_nvidia_gbm.json"
    mkdir -p "${pkgdir}/usr/lib/gbm"
    ln -sr "${pkgdir}/usr/lib/libnvidia-allocator.so.${pkgver}" "${pkgdir}/usr/lib/gbm/nvidia-drm_gbm.so"

    # firmware
    install -Dm644 -t "${pkgdir}/usr/lib/firmware/nvidia/${pkgver}/" firmware/*.bin

    # GLX extension module for X
    install -Dm755 "libglxserver_nvidia.so.${pkgver}" "${pkgdir}/usr/lib/nvidia/xorg/libglxserver_nvidia.so.${pkgver}"
    # Ensure that X finds glx
    ln -s "libglxserver_nvidia.so.${pkgver}" "${pkgdir}/usr/lib/nvidia/xorg/libglxserver_nvidia.so.1"
    ln -s "libglxserver_nvidia.so.${pkgver}" "${pkgdir}/usr/lib/nvidia/xorg/libglxserver_nvidia.so"

    install -Dm755 "libGLX_nvidia.so.${pkgver}" "${pkgdir}/usr/lib/libGLX_nvidia.so.${pkgver}"

    # OpenGL libraries
    install -Dm755 "libEGL_nvidia.so.${pkgver}" "${pkgdir}/usr/lib/libEGL_nvidia.so.${pkgver}"
    install -Dm755 "libGLESv1_CM_nvidia.so.${pkgver}" "${pkgdir}/usr/lib/libGLESv1_CM_nvidia.so.${pkgver}"
    install -Dm755 "libGLESv2_nvidia.so.${pkgver}" "${pkgdir}/usr/lib/libGLESv2_nvidia.so.${pkgver}"
    install -Dm644 "10_nvidia.json" "${pkgdir}/usr/share/glvnd/egl_vendor.d/10_nvidia.json"

    # OpenGL core library
    install -Dm755 "libnvidia-glcore.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-glcore.so.${pkgver}"
    install -Dm755 "libnvidia-eglcore.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-eglcore.so.${pkgver}"
    install -Dm755 "libnvidia-glsi.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-glsi.so.${pkgver}"

    # misc
    install -Dm755 "libnvidia-api.so.1" "${pkgdir}/usr/lib/libnvidia-api.so.1"
    install -Dm755 "libnvidia-fbc.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-fbc.so.${pkgver}"
    install -Dm755 "libnvidia-encode.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-encode.so.${pkgver}"
    install -Dm755 "libnvidia-cfg.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-cfg.so.${pkgver}"
    install -Dm755 "libnvidia-ml.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-ml.so.${pkgver}"
    install -Dm755 "libnvidia-glvkspirv.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-glvkspirv.so.${pkgver}"
    install -Dm755 "libnvidia-allocator.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-allocator.so.${pkgver}"
    install -Dm755 "libnvidia-vulkan-producer.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-vulkan-producer.so.${pkgver}"
    # Sigh libnvidia-vulkan-producer.so has no SONAME set so create_links doesn't catch it. NVIDIA please fix!
    patchelf --set-soname "libnvidia-vulkan-producer.so.1" "${pkgdir}/usr/lib/libnvidia-vulkan-producer.so.${pkgver}"
	
    # i hope snowman doesn't go angry for this
    install -Dm755 "libvgpucompat.so" "${pkgdir}/usr/lib/libvgpucompat.so"
	
    # Vulkan ICD
    install -Dm644 "nvidia_icd.json" "${pkgdir}/usr/share/vulkan/icd.d/nvidia_icd.json"
    install -Dm644 "nvidia_layers.json" "${pkgdir}/usr/share/vulkan/implicit_layer.d/nvidia_layers.json"

    # VDPAU
    install -Dm755 "libvdpau_nvidia.so.${pkgver}" "${pkgdir}/usr/lib/vdpau/libvdpau_nvidia.so.${pkgver}"

    # nvidia-tls library
    install -Dm755 "libnvidia-tls.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-tls.so.${pkgver}"

    # CUDA
    install -Dm755 "libcuda.so.${pkgver}" "${pkgdir}/usr/lib/libcuda.so.${pkgver}"
    install -Dm755 "libnvcuvid.so.${pkgver}" "${pkgdir}/usr/lib/libnvcuvid.so.${pkgver}"
    install -Dm755 "libcudadebugger.so.${pkgver}" "${pkgdir}/usr/lib/libcudadebugger.so.${pkgver}"

    # NVVM Compiler library loaded by the CUDA driver to do JIT link-time-optimization
    install -Dm644 "libnvidia-nvvm.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-nvvm.so.${pkgver}"

    # PTX JIT Compiler (Parallel Thread Execution (PTX) is a pseudo-assembly language for CUDA)
    install -Dm755 "libnvidia-ptxjitcompiler.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-ptxjitcompiler.so.${pkgver}"

    # raytracing
	install -Dm755 "nvoptix.bin" "${pkgdir}/usr/share/nvidia/nvoptix.bin"
    install -Dm755 "libnvoptix.so.${pkgver}" "${pkgdir}/usr/lib/libnvoptix.so.${pkgver}"
    install -Dm755 "libnvidia-rtcore.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-rtcore.so.${pkgver}"

    # NGX
    install -Dm755 nvidia-ngx-updater "${pkgdir}/usr/bin/nvidia-ngx-updater"
    install -Dm755 "libnvidia-ngx.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-ngx.so.${pkgver}"
    install -Dm755 _nvngx.dll "${pkgdir}/usr/lib/nvidia/wine/_nvngx.dll"
    install -Dm755 nvngx.dll "${pkgdir}/usr/lib/nvidia/wine/nvngx.dll"

    # Optical flow
    install -Dm755 "libnvidia-opticalflow.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-opticalflow.so.${pkgver}"

    # DEBUG
    install -Dm755 nvidia-debugdump "${pkgdir}/usr/bin/nvidia-debugdump"

    # nvidia-xconfig
    install -Dm755 nvidia-xconfig "${pkgdir}/usr/bin/nvidia-xconfig"
    install -Dm644 nvidia-xconfig.1.gz "${pkgdir}/usr/share/man/man1/nvidia-xconfig.1.gz"

    # nvidia-bug-report
    install -Dm755 nvidia-bug-report.sh "${pkgdir}/usr/bin/nvidia-bug-report.sh"

    # nvidia-smi
    install -Dm755 nvidia-smi "${pkgdir}/usr/bin/nvidia-smi"
    install -Dm644 nvidia-smi.1.gz "${pkgdir}/usr/share/man/man1/nvidia-smi.1.gz"

    # nvidia-cuda-mps
    install -Dm755 nvidia-cuda-mps-server "${pkgdir}/usr/bin/nvidia-cuda-mps-server"
    install -Dm755 nvidia-cuda-mps-control "${pkgdir}/usr/bin/nvidia-cuda-mps-control"
    install -Dm644 nvidia-cuda-mps-control.1.gz "${pkgdir}/usr/share/man/man1/nvidia-cuda-mps-control.1.gz"

    # nvidia-modprobe
    # This should be removed if nvidia fixed their uvm module!
    install -Dm4755 nvidia-modprobe "${pkgdir}/usr/bin/nvidia-modprobe"
    install -Dm644 nvidia-modprobe.1.gz "${pkgdir}/usr/share/man/man1/nvidia-modprobe.1.gz"

    # nvidia-persistenced
    install -Dm755 nvidia-persistenced "${pkgdir}/usr/bin/nvidia-persistenced"
    install -Dm644 nvidia-persistenced.1.gz "${pkgdir}/usr/share/man/man1/nvidia-persistenced.1.gz"
    install -Dm644 nvidia-persistenced-init/systemd/nvidia-persistenced.service.template "${pkgdir}/usr/lib/systemd/system/nvidia-persistenced.service"
    sed -i 's/__USER__/nvidia-persistenced/' "${pkgdir}/usr/lib/systemd/system/nvidia-persistenced.service"

    # application profiles
    install -Dm644 nvidia-application-profiles-${pkgver}-rc "${pkgdir}/usr/share/nvidia/nvidia-application-profiles-${pkgver}-rc"
    install -Dm644 nvidia-application-profiles-${pkgver}-key-documentation "${pkgdir}/usr/share/nvidia/nvidia-application-profiles-${pkgver}-key-documentation"

    # Licenses
    install -Dm644 LICENSE "${pkgdir}/usr/share/licenses/nvidia-utils/LICENSE"
    install -Dm644 GRID_LICENSE "${pkgdir}/usr/share/licenses/nvidia-utils/GRID_LICENSE"
    install -Dm644 grid-third-party-licenses.txt "${pkgdir}/usr/share/licenses/nvidia-utils/grid-third-party-licenses.txt"
    install -Dm644 README.txt "${pkgdir}/usr/share/doc/nvidia/README"
    install -Dm644 NVIDIA_Changelog "${pkgdir}/usr/share/doc/nvidia/NVIDIA_Changelog"
    cp -r html "${pkgdir}/usr/share/doc/nvidia/"
    ln -s nvidia "${pkgdir}/usr/share/doc/nvidia-utils"

    # new power management support
    install -Dm644 systemd/system/*.service -t "${pkgdir}/usr/lib/systemd/system"
    install -Dm755 systemd/system-sleep/nvidia "${pkgdir}/usr/lib/systemd/system-sleep/nvidia"
    install -Dm755 systemd/nvidia-sleep.sh "${pkgdir}/usr/bin/nvidia-sleep.sh"
    install -Dm755 nvidia-powerd "${pkgdir}/usr/bin/nvidia-powerd"
    install -Dm644 nvidia-dbus.conf "${pkgdir}"/usr/share/dbus-1/system.d/nvidia-dbus.conf

    # NVIDIA GRID
    install -Dm755 "libnvidia-vgpu.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-vgpu.so.${pkgver}"
    install -Dm755 "libnvidia-vgxcfg.so.${pkgver}" "${pkgdir}/usr/lib/libnvidia-vgxcfg.so.${pkgver}"
    install -Dm755 "$srcdir/libvgpu_unlock_rs.so" "${pkgdir}/usr/lib/nvidia/libvgpu_unlock_rs.so"
    install -Dm755 "nvidia-vgpud" "${pkgdir}/usr/bin/nvidia-vgpud"
    install -Dm755 "nvidia-vgpu-mgr" "${pkgdir}/usr/bin/nvidia-vgpu-mgr"
    install -Dm755 "sriov-manage" "${pkgdir}/usr/lib/nvidia/sriov-manage"
    install -Dm644 "vgpuConfig.xml" "${pkgdir}/usr/share/nvidia/vgpu/vgpuConfig.xml"
    install -Dm644 init-scripts/systemd/*.service -t "${pkgdir}/usr/lib/systemd/system"
    install -Dm644 "$srcdir/nvidia-grid.conf" "${pkgdir}/usr/share/dbus-1/system.d/nvidia-grid.conf"

    # distro specific files must be installed in /usr/share/X11/xorg.conf.d
    install -Dm644 "${srcdir}/nvidia-drm-outputclass.conf" "${pkgdir}/usr/share/X11/xorg.conf.d/10-nvidia-drm-outputclass.conf"
    install -Dm644 "${srcdir}/nvidia-utils.sysusers" "${pkgdir}/usr/lib/sysusers.d/$pkgname.conf"
    install -Dm644 "${srcdir}/nvidia.rules" "$pkgdir"/usr/lib/udev/rules.d/60-nvidia.rules

    # modprobe.d
    echo "blacklist nouveau" | install -Dm644 /dev/stdin "${pkgdir}/usr/lib/modprobe.d/${pkgname}.conf"
    echo "nvidia-uvm" | install -Dm644 /dev/stdin "${pkgdir}/usr/lib/modules-load.d/${pkgname}.conf"

    create_links
}
