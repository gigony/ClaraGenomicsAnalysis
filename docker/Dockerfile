#
# Copyright (c) 2019, NVIDIA CORPORATION.  All rights reserved.
#
# NVIDIA CORPORATION and its licensors retain all intellectual property
# and proprietary rights in and to this software, related documentation
# and any modifications thereto.  Any use, reproduction, disclosure or
# distribution of this software and related documentation without an express
# license agreement from NVIDIA CORPORATION is strictly prohibited.
#

ARG FROM_IMAGE=nvidia/cuda
ARG CUDA_VERSION=10.1
ARG LINUX_VERSION=ubuntu18.04

FROM ${FROM_IMAGE}:${CUDA_VERSION}-devel-${LINUX_VERSION} AS build

ARG CC_VERSION=7
ARG CXX_VERSION=7
ARG CMAKE_VERSION=3.14.7
ARG HTSLIB_VERSION=1.9
ARG CGA_VERSION=0.3.0
ARG RACON_GPU_VERSION=integrate-cga-v0.3.0

ENV CC_VERSION=${CC_VERSION}
ENV CXX_VERSION=${CC_VERSION}
ENV CMAKE_VERSION=${CMAKE_VERSION}
ENV HTSLIB_VERSION=${HTSLIB_VERSION}
ENV CGA_VERSION=${CGA_VERSION}
ENV RACON_GPU_VERSION=${RACON_GPU_VERSION}

# Ensure apt-get won't prompt for selecting options
ENV DEBIAN_FRONTEND=noninteractive
ENV LC_ALL=C.UTF-8
ENV LANG=C.UTF-8

# Environment variables for compilers
ENV LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda/lib64:/usr/local/lib
ENV CUDA_HOME=/usr/local/cuda
ENV CC=/usr/bin/gcc-${CC_VERSION}
ENV CXX=/usr/bin/g++-${CXX_VERSION}
ENV CUDAHOSTCXX=/usr/bin/g++-${CXX_VERSION}

# Install dependencies for htslib and Clara Genomics SDK
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        git \
        gcc-${CC_VERSION} \
        g++-${CXX_VERSION} \
        python3 \
        python3-pip \
        autoconf \
        automake \
        make \
        perl \
        zlib1g-dev \
        libbz2-dev \
        liblzma-dev \
        libcurl4-gnutls-dev \
        libssl-dev \
        wget \
        rsync \
        python3-dev \
    && rm -rf /var/lib/apt/lists/* \
    && rm -f /usr/bin/python /usr/bin/pip \
    && ln -s /usr/bin/python3 /usr/bin/python \
    && ln -s /usr/bin/pip3 /usr/bin/pip

WORKDIR /opt/nvidia/clara-genomics/build

# Install htslib
RUN wget https://github.com/samtools/htslib/releases/download/${HTSLIB_VERSION}/htslib-${HTSLIB_VERSION}.tar.bz2 \
    && tar xvf htslib-${HTSLIB_VERSION}.tar.bz2 \
    && cd htslib-${HTSLIB_VERSION} \
    && ./configure \
    && make -j16 prefix=install install \
    && rsync -avz install/* /usr/local/

# Install cmake
RUN wget -O- "https://github.com/Kitware/CMake/releases/download/v${CMAKE_VERSION}/cmake-${CMAKE_VERSION}-Linux-x86_64.tar.gz" \
    | tar --strip-components=1 -xz -C /usr/local

RUN git clone --recursive -b ${CGA_VERSION} --single-branch https://github.com/clara-genomics/ClaraGenomicsAnalysis.git
RUN cd ClaraGenomicsAnalysis/pyclaragenomics \
    && pip install --no-cache-dir --upgrade wheel setuptools \
    && pip install --no-cache-dir -r requirements.txt \
    && python setup.py sdist bdist_wheel

RUN git clone --recursive -b ${RACON_GPU_VERSION} --single-branch https://github.com/clara-genomics/racon-gpu.git
RUN cd racon-gpu \
    && mkdir build \
    && cd build \
    && cmake -DCMAKE_BUILD_TYPE=Release -Dracon_build_tests=ON -Dracon_build_wrapper=ON -Dracon_enable_cuda=ON .. \
    && make


FROM ${FROM_IMAGE}:${CUDA_VERSION}-base-${LINUX_VERSION}

# Ensure apt-get won't prompt for selecting options
ENV DEBIAN_FRONTEND=noninteractive
ENV LC_ALL=C.UTF-8
ENV LANG=C.UTF-8

# Set environment variables
ENV CUDA_HOME=/usr/local/cuda
ENV PATH=/opt/nvidia/clara-genomics/bin:/opt/nvidia/clara-genomics/racon-gpu/bin:${PATH}
ENV LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda/lib64:/usr/local/lib:/opt/nvidia/clara-genomics/lib

WORKDIR /workspace

# Copy Clara Genomics Analysis binaries
COPY --from=build /opt/nvidia/clara-genomics/build/ClaraGenomicsAnalysis/pyclaragenomics/requirements.txt /opt/nvidia/clara-genomics/pyclaragenomics/
COPY --from=build /opt/nvidia/clara-genomics/build/ClaraGenomicsAnalysis/pyclaragenomics/dist/*.whl /opt/nvidia/clara-genomics/pyclaragenomics/dist/
COPY --from=build /opt/nvidia/clara-genomics/build/ClaraGenomicsAnalysis/pyclaragenomics/cga_build/install/ /opt/nvidia/clara-genomics/
# Copy Racon GPU binaries
COPY --from=build /opt/nvidia/clara-genomics/build/racon-gpu/build/bin/ /opt/nvidia/clara-genomics/racon-gpu/bin/
COPY --from=build /opt/nvidia/clara-genomics/build/racon-gpu/build/vendor/rampler/bin/ /opt/nvidia/clara-genomics/racon-gpu/bin/
COPY --from=build /opt/nvidia/clara-genomics/build/racon-gpu/test/data/ /opt/nvidia/clara-genomics/build/racon-gpu/test/data/
# Copy htslib binaries
COPY --from=build /opt/nvidia/clara-genomics/build/htslib-1.9/install/ /usr/local/

# Correct binary locations for racon_wrapper
RUN sed -i -e 's#/opt/nvidia/clara-genomics/build/racon-gpu/build/bin/racon#/opt/nvidia/clara-genomics/racon-gpu/bin/racon#' \
        -e 's#/opt/nvidia/clara-genomics/build/racon-gpu/build/vendor/rampler/bin/rampler#/opt/nvidia/clara-genomics/racon-gpu/bin/rampler#' \
        /opt/nvidia/clara-genomics/racon-gpu/bin/racon_wrapper

# Install dependencies for Clara Genomics SDK
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        python3 \
        python3-pip \
        libcurl4 \
        libcurl3-gnutls \
    && rm -rf /var/lib/apt/lists/* \
    && rm -f /usr/bin/python /usr/bin/pip \
    && ln -s /usr/bin/python3 /usr/bin/python \
    && ln -s /usr/bin/pip3 /usr/bin/pip \
    && pip install --no-cache-dir --upgrade wheel setuptools \
    && pip install --no-cache-dir -r /opt/nvidia/clara-genomics/pyclaragenomics/requirements.txt \
    && pip install --no-cache-dir /opt/nvidia/clara-genomics/pyclaragenomics/dist/*.whl \
    && rm -rf /var/lib/apt/lists/*
