FROM debian:bullseye-slim as builder
ARG DOCKER_TAG
ARG BUILD_CONCURRENCY
RUN mkdir -p /src  && mkdir -p /opt

RUN apt-get update && \
    apt-get -y --no-install-recommends install ca-certificates cmake make git gcc g++ libbz2-dev libxml2-dev wget \
    libzip-dev libboost1.74-all-dev lua5.4 liblua5.4-dev pkg-config -o APT::Install-Suggests=0 -o APT::Install-Recommends=0

RUN NPROC=${BUILD_CONCURRENCY:-$(nproc)} && \
    ldconfig /usr/local/lib && \
    git clone --branch v2021.3.0 --single-branch https://github.com/oneapi-src/oneTBB.git && \
    cd oneTBB && \
    mkdir build && \
    cd build && \
    cmake -DTBB_TEST=OFF -DCMAKE_BUILD_TYPE=Release ..  && \
    cmake --build . && \
    cmake --install .

COPY . /src
WORKDIR /src

RUN NPROC=${BUILD_CONCURRENCY:-$(nproc)} && \
    echo "Building OSRM ${DOCKER_TAG}" && \
    git show --format="%H" | head -n1 > /opt/OSRM_GITSHA && \
    echo "Building OSRM gitsha $(cat /opt/OSRM_GITSHA)" && \
    mkdir -p build && \
    cd build && \
    BUILD_TYPE="Release" && \
    ENABLE_ASSERTIONS="Off" && \
    BUILD_TOOLS="Off" && \
    case ${DOCKER_TAG} in *"-debug"*) BUILD_TYPE="Debug";; esac && \
    case ${DOCKER_TAG} in *"-assertions"*) BUILD_TYPE="RelWithDebInfo" && ENABLE_ASSERTIONS="On" && BUILD_TOOLS="On";; esac && \
    echo "Building ${BUILD_TYPE} with ENABLE_ASSERTIONS=${ENABLE_ASSERTIONS} BUILD_TOOLS=${BUILD_TOOLS}" && \
    cmake .. -DCMAKE_BUILD_TYPE=${BUILD_TYPE} -DENABLE_ASSERTIONS=${ENABLE_ASSERTIONS} -DBUILD_TOOLS=${BUILD_TOOLS} -DENABLE_LTO=On && \
    make -j${NPROC} install && \
    cd ../profiles && \
    cp -r * /opt && \
    strip /usr/local/bin/* && \
    rm -rf /src


# Multistage build to reduce image size - https://docs.docker.com/engine/userguide/eng-image/multistage-build/#use-multi-stage-builds
# Only the content below ends up in the image, this helps remove /src from the image (which is large)
FROM debian:bullseye-slim as runstage

COPY --from=builder /usr/local /usr/local
COPY --from=builder /opt /opt

RUN apt-get update && \
    apt-get install -y --no-install-recommends libboost-program-options1.74.0 libboost-regex1.74.0 \
        libboost-date-time1.74.0 libboost-chrono1.74.0 libboost-filesystem1.74.0 \
        libboost-iostreams1.74.0 libboost-system1.74.0 libboost-thread1.74.0 \
        expat liblua5.4-0 && \
#    rm -rf /var/lib/apt/lists/* && \
# add /usr/local/lib to ldconfig to allow loading libraries from there
    ldconfig /usr/local/lib

RUN /usr/local/bin/osrm-extract --help && \
    /usr/local/bin/osrm-routed --help && \
    /usr/local/bin/osrm-contract --help && \
    /usr/local/bin/osrm-partition --help && \
    /usr/local/bin/osrm-customize --help

WORKDIR /opt

EXPOSE 5000
