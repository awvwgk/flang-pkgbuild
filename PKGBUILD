_compiler=clang
pkgname=flang
pkgver=13.0.0
pkgrel=1
pkgdesc="Fortran frontend for LLVM"
arch=('x86_64')
url="https://flang.llvm.org/"
license=("custom:Apache 2.0 with LLVM Exception")
depends=("clang=${pkgver}"
         "llvm=${pkgver}"
         "mlir=${pkgver}")
makedepends=("clang-analyzer=${pkgver}"
             "clang-tools-extra=${pkgver}"
             "cmake>=3.4.3"
             "lld"
             "ninja"
             "pkg-config"
             "polly=${pkgver}"
             "gawk"
             $([[ "$_compiler" == "clang" ]] && echo \
               "compiler-rt" \
               "libc++")
             $([[ "$_compiler" == "gcc" ]] && echo \
               "gcc")
             )
options=('!debug' 'strip')
source=("https://github.com/llvm/llvm-project/releases/download/llvmorg-${pkgver}/flang-${pkgver}.src.tar.xz"
        "0001-cast-cxx11-narrowing.patch"
        "0002-cmake-22162.patch"
        "0003-Link-against-libclang-cpp.so.patch")
sha256sums=('13bc580342bec32b6158c8cddeb276bd428d9fc8fd23d13179c8aa97bbba37d5'
            'f8babd240968599597cf2b906fad44b62e2d92ac7ea144530a3f9542e96e06b1'
            '77fb0612217b6af7a122f586a9d0d334cd48bb201509bf72e8f8e6244616e895'
            '8cb7fb58455b3d180538ca3522a6e3d6d9ec3ee66d7f339c2ccae82a7302bd2a')

apply_patch_with_msg() {
  for _patch in "$@"
  do
    msg2 "Applying ${_patch}"
    patch -Nbp1 -i "${srcdir}/${_patch}"
  done
}

prepare() {
  cd "${srcdir}/flang-${pkgver}.src"

  apply_patch_with_msg \
    "0001-cast-cxx11-narrowing.patch" \
    "0002-cmake-22162.patch" \
    "0003-Link-against-libclang-cpp.so.patch"
}

build() {
  cd "${srcdir}"

  case "${CARCH}" in
    i?86|armv7)
      # lld needs all the address space it can get.
      LDFLAGS+=" -Wl,--large-address-aware"
      ;;
  esac

  local -a extra_args

  if check_option "debug" "y"; then
    extra_args+=(-DCMAKE_BUILD_TYPE=Debug)
  else
    extra_args+=(-DCMAKE_BUILD_TYPE=Release)
  fi

  if [ "${_compiler}" == "clang" ]; then
    export CC='clang'
    export CXX='clang++'
    extra_args+=(-DLLVM_ENABLE_LLD=ON)
  fi

  # try to figure out a reasonable CMAKE_BUILD_PARALLEL_LEVEL
  if [ -z "${CMAKE_BUILD_PARALLEL_LEVEL+x}" ]; then
    # figure about 1 job per 2GB RAM
    local _jobs=$(awk '{ if ($1 == "MemTotal:") { printf "%.0f", $2/(2*1024*1024) } }' /proc/meminfo)
    # EXCEPT on GHA, which is being really difficult.
    if (( _jobs < 1 )) || [ -n "${CI+x}" ]; then
      _jobs=1
    elif (( _jobs > $(nproc) + 2 )); then
      _jobs=$(( $(nproc) + 2 ))
    fi
    export CMAKE_BUILD_PARALLEL_LEVEL=${_jobs}
  fi

  [[ -d build-${CARCH} ]] && rm -rf build-${CARCH}
  mkdir build-${CARCH} && cd build-${CARCH}

  cmake \
    -GNinja \
    -Wno-dev \
    -DCLANG_DIR=/usr/lib/cmake/clang \
    -DCMAKE_INSTALL_PREFIX=/usr \
    -DLLVM_ENABLE_ASSERTIONS=OFF \
    -DLLVM_ENABLE_THREADS=ON \
    -DLLVM_HOST_TRIPLE="${CARCH}-pc-linux-gnu" \
    -DLLVM_LINK_LLVM_DYLIB=ON \
    "${extra_args[@]}" \
    ../flang-${pkgver}.src

  ln -s $(which mlir-tblgen) include/flang/Optimizer/CodeGen/mlir-tblgen
  ln -s $(which mlir-tblgen) include/flang/Optimizer/Dialect/mlir-tblgen
  ln -s $(which mlir-tblgen) include/flang/Optimizer/Transforms/mlir-tblgen
  cmake --build .
}

check() {
  cd "${srcdir}"/build-${CARCH}
  cmake --build . -- check-flang || true
}

package() {
  DESTDIR="${pkgdir}" cmake --install "${srcdir}/build-${CARCH}"
}
