#   v2.0.0 builds kspaceFirstOrder-CUDA from source
#   (https://github.com/klepo/k-Wave-Fluid-CUDA) using CUDA 12.2 and targets
#   sm_75 (Turing) through sm_90 (Hopper), covering all current consumer and
#   data-centre NVIDIA GPUs.

# ======================== Build stage ========================
FROM nvidia/cuda:12.2.2-devel-ubuntu22.04 AS build

LABEL maintainer=lmoyasans

ENV SC_BUILD_TARGET=build

ENV KWAVE_COMMIT=44ca4ed1fce2319c1558ec5ba3a165d3d44a5da1

RUN apt-get update \
 && apt-get -y install --no-install-recommends \
    git \
    make \
    g++ \
    libhdf5-dev \
    zlib1g-dev \
    libaec-dev \
 && rm -rf /var/lib/apt/lists/*

# Create the HDF5 prefix layout expected by the k-Wave Makefile:
#   $(HDF5_DIR)/include  and  $(HDF5_DIR)/lib
# Ubuntu 22.04 puts the serial-variant files in non-standard sub-directories.
RUN mkdir -p /opt/hdf5 \
 && ln -s /usr/include/hdf5/serial       /opt/hdf5/include \
 && ln -s /usr/lib/x86_64-linux-gnu/hdf5/serial /opt/hdf5/lib

# libaec-dev provides SZIP compatibility; ensure libsz.a is reachable.
RUN ls /usr/lib/x86_64-linux-gnu/libsz.a 2>/dev/null \
 || ln -s /usr/lib/x86_64-linux-gnu/libaec.a \
          /usr/lib/x86_64-linux-gnu/libsz.a

WORKDIR /build

# Clone the open-source k-Wave CUDA fluid solver (pinned to known-good commit).
RUN git clone https://github.com/klepo/k-Wave-Fluid-CUDA.git . \
 && git checkout ${KWAVE_COMMIT}

# Patch source to fix compilation issues in the feat_q_computing branch:
#
# 1. Case mismatch: sources include <KSpaceSolver/SolverCUDAKernels.cuh> (all-caps)
#    but the actual file is SolverCudaKernels.cuh — create a symlink.
# 2. KSpaceFirstOrderSolver.cpp uses std::ifstream (<fstream>), strcmp (<cstring>),
#    sysconf/_SC_PHYS_PAGES/_SC_PAGE_SIZE (<unistd.h>/<sys/sysinfo.h>),
#    and _mm_malloc/_mm_free (<xmmintrin.h>) without including the required headers.
# 3. IndexMatrix.cpp uses ceilf() without #include <cmath>.
# 4. CompressHelper.cpp uses std::numeric_limits without #include <limits>.
RUN ln -s SolverCudaKernels.cuh KSpaceSolver/SolverCUDAKernels.cuh && \
    sed -i 's|^#include <ctime>|#include <ctime>\n#include <fstream>\n#include <cstring>\n#include <unistd.h>\n#include <xmmintrin.h>|' \
        KSpaceSolver/KSpaceFirstOrderSolver.cpp && \
    sed -i '1s|^|#include <cmath>\n|' MatrixClasses/IndexMatrix.cpp && \
    sed -i '1s|^|#include <limits>\n|' Compression/CompressHelper.cpp

# Build with CUDA 12.2.
# SEMI linking: HDF5 + zlib + szip linked statically, cuFFT dynamically
# (avoids the static-cuFFT relocation bug noted in the upstream Makefile).
# CUDA_ARCH: cover Turing (sm_75) → Ada Lovelace (sm_89) → Hopper (sm_90).
# LIBS is passed on the command line to override the ifeq(LINKING,SEMI) block,
# since Ubuntu 22.04 places static libs in /usr/lib/x86_64-linux-gnu/ not /usr/lib/.
RUN make -f Makefiles/Release/Makefile -j"$(nproc)" \
    HDF5_DIR=/opt/hdf5 \
    CUDA_DIR=/usr/local/cuda \
    LINKING=SEMI \
    LIBS="/opt/hdf5/lib/libhdf5_hl.a \
          /opt/hdf5/lib/libhdf5.a \
          /usr/lib/x86_64-linux-gnu/libz.a \
          /usr/lib/x86_64-linux-gnu/libsz.a \
          /usr/lib/x86_64-linux-gnu/libaec.a \
          -lcufft -ldl" \
    CUDA_ARCH="--generate-code arch=compute_75,code=sm_75 \
               --generate-code arch=compute_80,code=sm_80 \
               --generate-code arch=compute_86,code=sm_86 \
               --generate-code arch=compute_89,code=sm_89 \
               --generate-code arch=compute_90,code=sm_90"

RUN mkdir -p /build/bin \
 && cp kspaceFirstOrder-CUDA /build/bin/

# ======================== Production stage ========================
# simcore-user uid=8004(${SC_USER_NAME}) gid=8004(${SC_USER_NAME}) groups=8004(${SC_USER_NAME})
FROM nvidia/cuda:12.2.2-runtime-ubuntu22.04 AS production

ENV SC_USER_ID=8004 \
    SC_USER_NAME=scu

RUN adduser --uid ${SC_USER_ID} --disabled-password --gecos "" \
        --shell /bin/sh --home /home/${SC_USER_NAME} ${SC_USER_NAME}

ENV SC_BUILD_TARGET=production \
    SC_BOOT_MODE=production

ENV INPUT_FOLDER="/input" \
    OUTPUT_FOLDER="/output"

# GPU access: all modern NVIDIA GPUs are visible.
# Ada Lovelace (sm_89) and Hopper (sm_90) are fully supported.
# Requires CUDA-capable driver ≥ 525 (ships CUDA 12.0 user-space ABI).
ENV NVIDIA_VISIBLE_DEVICES=all \
    NVIDIA_DRIVER_CAPABILITIES=compute,utility \
    NVIDIA_REQUIRE_CUDA="cuda>=12.0 driver>=525"

WORKDIR /home/${SC_USER_NAME}

RUN apt-get update \
 && apt-get -y install --no-install-recommends \
    jq \
    libgomp1 \
 && rm -rf /var/lib/apt/lists/*

# copy docker bootup scripts
COPY --chown=${SC_USER_NAME}:${SC_USER_NAME} docker/custom/*.sh docker/
# copy simcore service cli
COPY --chown=${SC_USER_NAME}:${SC_USER_NAME} service.cli/ service.cli/
# necessary to be able to call run directly without sh in front
ENV PATH="/home/${SC_USER_NAME}/service.cli:${PATH}"

# copy binary from build stage
COPY --from=build --chown=${SC_USER_NAME}:${SC_USER_NAME} /build/bin k_wave
RUN chmod +x k_wave/*

ENTRYPOINT [ "/bin/sh", "docker/entrypoint.sh" ]
CMD ["run"]
