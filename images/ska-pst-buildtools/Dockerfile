FROM library/ubuntu:22.04 as base

ENV DEBIAN_FRONTEND=noninteractive
ENV LD_LIBRARY_PATH=/usr/local/lib/

ARG DEPENDENCIES_PATH=dependencies/ska-pst-buildtools

# Copy installation payloads
# COPY . /mnt/ska-pst-buildtools
COPY ./dependencies /mnt/ska-pst-buildtools/dependencies/
COPY Makefile /mnt/ska-pst-buildtools
COPY .pst /mnt/ska-pst-buildtools/.pst/
COPY .make /mnt/ska-pst-buildtools/.make/
COPY ./resources/ /mnt/ska-pst-buildtools/resources/
COPY ./images/ska-pst-buildtools/ctest2junit.xsl /usr/local/share/ctest2junit/
COPY ./images/ska-pst-buildtools/ctest2junit /usr/local/bin/

WORKDIR /mnt/ska-pst-buildtools

# Prepare environment
RUN apt-get update -y \
    && apt-get install -y make

# Install common apt dependencies
ARG PKG_CLI_PAYLOAD=${DEPENDENCIES_PATH}/apt.txt
ARG PKG_CLI_CMD=apt-get
ARG PKG_CLI_PARAMETERS='install --no-install-recommends -y'
RUN stat ${PKG_CLI_PAYLOAD} \
    && PKG_CLI_PAYLOAD=${PKG_CLI_PAYLOAD} PKG_CLI_PARAMETERS=${PKG_CLI_PARAMETERS} make local-pkg-install

# Install common pip dependencies
ARG PIP_CLI_PAYLOAD=${DEPENDENCIES_PATH}/pip.txt
RUN stat ${PIP_CLI_PAYLOAD} \
    && PIP_CLI_PAYLOAD=${PIP_CLI_PAYLOAD} make local-pip-install

# Install cmake
# Use cmake's install script rather than build.
ARG CMAKE_VERSION=3.24.0-rc1
ARG CMAKE_RELEASE_URL=https://github.com/Kitware/CMake/releases/download/v${CMAKE_VERSION}
ARG CMAKE_FILE=cmake-${CMAKE_VERSION}-linux-x86_64.sh

WORKDIR /mnt/cmake
RUN curl -L -O ${CMAKE_RELEASE_URL}/${CMAKE_FILE} \
    && sh ${CMAKE_FILE} -- --skip-license --prefix=/usr/local

# Prepare environment for PROTOBUF
ARG PROTOBUF_VERSION=21.9
ARG PROTOBUF_FULL_VERSION=3.${PROTOBUF_VERSION}
ARG PROTOBUF_RELEASE_URL=https://github.com/protocolbuffers/protobuf/releases/download/v${PROTOBUF_VERSION}
ARG PROTOBUF_FILE=protobuf-cpp-${PROTOBUF_FULL_VERSION}.tar.gz
ARG PROTOBUF_FILE_FOLDER=protobuf-${PROTOBUF_FULL_VERSION}
ARG PROTOBUF=protobuf
WORKDIR /mnt/${PROTOBUF}

# Download PROTOBUF source code
RUN curl -L -O ${PROTOBUF_RELEASE_URL}/${PROTOBUF_FILE} \
    && tar -xzvf ${PROTOBUF_FILE} \
    && stat /mnt/${PROTOBUF}/${PROTOBUF_FILE_FOLDER}

# Compile PROTOBUF source code
# https://github.com/protocolbuffers/protobuf/blob/main/src/README.md#c-installation---unix
WORKDIR /mnt/${PROTOBUF}/${PROTOBUF_FILE_FOLDER}
RUN ./configure --prefix=/usr/local \
    && make -j$(nproc) && make install -j$(nproc) \
    && protoc --version

ARG GOLANG=golang-go
ARG GOLANG_VERSION=1.19.3
ARG GOLANG_FILE=go${GOLANG_VERSION}.linux-amd64.tar.gz
ARG GOLANG_RELEASE_FILE=https://go.dev/dl/${GOLANG_FILE}
WORKDIR /mnt/${GOLANG}-${GOLANG_VERSION}
RUN rm -rf /usr/local/go \
    && curl -L -O ${GOLANG_RELEASE_FILE} \
    && tar -C /usr/local -xzf ${GOLANG_FILE}

ENV PATH $PATH:/usr/local/go/bin
ENV GOBIN /usr/local/bin

ARG PROTOC_GEN_DOC_VERSION=1.5.1
ARG PROTOC_GEN_DOC_RELEASE_URL=https://github.com/pseudomuto/protoc-gen-doc/archive/refs/tags
ARG PROTOC_GEN_DOC_RELEASE_FILE=v${PROTOC_GEN_DOC_VERSION}.tar.gz
ARG PROTOC_GEN_DOC_FILE_FOLDER=protoc-gen-doc-${PROTOC_GEN_DOC_VERSION}
ARG PROTOC_GEN_DOC=protoc-gen-doc

WORKDIR /mnt/${PROTOC_GEN_DOC}
RUN curl -L -O ${PROTOC_GEN_DOC_RELEASE_URL}/${PROTOC_GEN_DOC_RELEASE_FILE} \
    && tar -xvf ${PROTOC_GEN_DOC_RELEASE_FILE} \
    && stat ${PROTOC_GEN_DOC_FILE_FOLDER} \
    && cd ${PROTOC_GEN_DOC_FILE_FOLDER} \
    && go install ./cmd/protoc-gen-doc \
    && /usr/local/bin/protoc-gen-doc -version

# Compile gRPC
# git clone --recurse-submodules -b v1.50.0 --depth 1 --shallow-submodules https://github.com/grpc/grpc

ARG GRPC_VERSION=1.51.0
ARG GRPC_REPOSITORY=https://github.com/grpc/grpc
# ARG GRPC_RELEASE_FILE=v${GRPC_VERSION}.tar.gz
ARG GRPC_FILE_FOLDER=grpc-${GRPC_VERSION}
ARG GRPC=grpc

WORKDIR /mnt/${GRPC}
RUN git clone --recurse-submodules -b v${GRPC_VERSION} --depth 1 \
    --shallow-submodules ${GRPC_REPOSITORY} ${GRPC_FILE_FOLDER}

WORKDIR /mnt/${GRPC}-build
RUN cmake -DgRPC_INSTALL=ON -DgRPC_PROTOBUF_PROVIDER=package  \
	  -DBUILD_SHARED_LIBS=ON \
      -DgRPC_BUILD_TESTS=OFF \
      -DCMAKE_INSTALL_PREFIX=/usr/local \
      /mnt/${GRPC}/${GRPC_FILE_FOLDER} \
    && make -j$(nproc) && make install -j$(nproc)

# Prepare environment for PSRDADA
ARG PSRDADA=psrdada-code
WORKDIR /mnt/${PSRDADA}-build

# Compile PSRDADA without IBVERBS and RDMACM
RUN cmake /mnt/ska-pst-buildtools/resources/${PSRDADA} -DCMAKE_DISABLE_FIND_PACKAGE_rdmacm=TRUE -DCMAKE_DISABLE_FIND_PACKAGE_ibverbs=TRUE -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF \
    && make -j$(nproc) \
    && make install \
    && cp $(pwd)/Management/psrdada.pc ${LD_LIBRARY_PATH}/pkgconfig/

ARG SPDLOG_VERSION=1.11.0
ARG SPDLOG_RELEASE_URL=https://github.com/gabime/spdlog/archive/refs/tags/
ARG SPDLOG_RELEASE_FILE=v${SPDLOG_VERSION}.tar.gz
ARG SPDLOG_FILE_FOLDER=spdlog-${SPDLOG_VERSION}
ARG SPDLOG=spdlog

WORKDIR /mnt/${SPDLOG}
RUN curl -L -O ${SPDLOG_RELEASE_URL}/${SPDLOG_RELEASE_FILE} \
    && tar -xvf ${SPDLOG_RELEASE_FILE} \
    && stat ${SPDLOG_FILE_FOLDER}

WORKDIR /mnt/${SPDLOG}/${SPDLOG_FILE_FOLDER}
COPY ./src/spdlog/include/tweakme.h ./include/spdlog/tweakme.h

WORKDIR /mnt/${SPDLOG}-build
RUN cmake /mnt/${SPDLOG}/${SPDLOG_FILE_FOLDER} \
      -DCMAKE_INSTALL_PREFIX=/usr/local \
      -DCMAKE_POSITION_INDEPENDENT_CODE=on \
    && make -j$(nproc) \
    && make install


ARG GOOGLETEST_VERSION=1.12.1
ARG GOOGLETEST_RELEASE_URL=https://github.com/google/googletest/archive/refs/tags
ARG GOOGLETEST_RELEASE_FILE=release-${GOOGLETEST_VERSION}.tar.gz
ARG GOOGLETEST_FILE_FOLDER=googletest-release-${GOOGLETEST_VERSION}
ARG GOOGLETEST=googletest

WORKDIR /mnt/${GOOGLETEST}
RUN curl -L -O ${GOOGLETEST_RELEASE_URL}/${GOOGLETEST_RELEASE_FILE} \
    && stat ${GOOGLETEST_RELEASE_FILE} \
    && tar -xvf ${GOOGLETEST_RELEASE_FILE} \
    && stat ${GOOGLETEST_FILE_FOLDER}

WORKDIR /mnt/${GOOGLETEST}-build
RUN cmake /mnt/${GOOGLETEST}/${GOOGLETEST_FILE_FOLDER} \
      -DCMAKE_INSTALL_PREFIX=/usr/local \
    && make -j$(nproc) \
    && make install

# Remove cached pyyaml info file
RUN rm -rf /usr/lib/python3/dist-packages/PyYAML.*
