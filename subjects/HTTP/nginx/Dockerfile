FROM ubuntu:20.04

# Install common dependencies
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get -y update && \
    apt-get -y install sudo \
    apt-utils \
    build-essential \
    openssl \
    clang \
    graphviz-dev \
    git \
    autoconf \
    libgnutls28-dev \
    llvm \
    python3-pip \
    nano \
    net-tools \
    vim \
    gdb \
    netcat \
    strace \
    wget \
    libpcre3 \
    libpcre3-dev \
    zlib1g-dev \
    libssl-dev

RUN chmod 777 /tmp

RUN pip3 install gcovr==4.2

WORKDIR /root

# Import environment variable to pass as parameter to make (e.g., to make parallel builds with -j)
ARG MAKE_OPT

# Set up fuzzers
RUN git clone https://github.com/nrauschcom/aflnet.git && \
    cd aflnet && \
    make clean all $MAKE_OPT && \
    cd llvm_mode && make $MAKE_OPT

RUN git clone https://github.com/profuzzbench/aflnwe.git && \
    cd aflnwe && \
    make clean all $MAKE_OPT && \
    cd llvm_mode && make $MAKE_OPT

# Set up environment variables for AFLNet
ENV WORKDIR="/root/experiments"
ENV AFLNET="/root/aflnet"
ENV PATH="${PATH}:${AFLNET}:/root/.local/bin:${WORKDIR}"
ENV AFL_PATH="${AFLNET}"
ENV AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1 \
    AFL_SKIP_CPUFREQ=1 \
    AFL_NO_AFFINITY=1

RUN mkdir $WORKDIR

RUN apt-get -y update && \
    apt-get -y install \
    bison \
    flex \
    autotools-dev autoconf automake libtool gettext gawk \
    gperf antlr3 libantlr3c-dev libconfuse-dev libunistring-dev libsqlite3-dev \
    libavcodec-dev libavformat-dev libavfilter-dev libswscale-dev libavutil-dev \
    libasound2-dev libmxml-dev libgcrypt20-dev libavahi-client-dev zlib1g-dev \
    libevent-dev libplist-dev libsodium-dev libjson-c-dev libwebsockets-dev \
    libcurl4-openssl-dev avahi-daemon

# Set up environment variables for ASAN
ENV ASAN_OPTIONS='abort_on_error=1:symbolize=0:detect_leaks=0:detect_stack_use_after_return=1:detect_container_overflow=0:poison_array_cookie=0:malloc_fill_byte=0:max_malloc_fill_size=16777216'


RUN cd $WORKDIR && \
    wget https://nginx.org/download/nginx-1.24.0.tar.gz && \
    tar zxf nginx-1.24.0.tar.gz && \
    cd nginx-1.24.0 && \
    CC=$AFLNET/afl-clang-fast ./configure --prefix=/usr && \
    make $MAKE_OPT

RUN cd $WORKDIR && \
    wget https://nginx.org/download/nginx-1.24.0.tar.gz && \
    mkdir nginx-1.24.0-gcov && \
    mv nginx-1.24.0.tar.gz nginx-1.24.0-gcov && \
    cd nginx-1.24.0-gcov && \
    tar zxf nginx-1.24.0.tar.gz --strip-components 1 && \
    CFLAGS="-fprofile-arcs -ftest-coverage" LDFLAGS="-fprofile-arcs -ftest-coverage" ./configure --prefix=/usr --with-cc-opt='-fprofile-arcs -ftest-coverage --coverage' --with-ld-opt=-lgcov && \
    make $MAKE_OPT

RUN mkdir -p /usr/logs && \
    touch /usr/logs/error.log && \
    touch /usr/logs/access.log && \
    touch /usr/logs/nginx.pid

RUN mkdir -p /usr/conf && \
    cp -r $WORKDIR/nginx-1.24.0/conf/* /usr/conf && \
    rm /usr/conf/nginx.conf

RUN mkdir /var/www/ && \
    touch /var/www/index.html

COPY in-http ${WORKDIR}/in-http
COPY cov_script.sh ${WORKDIR}/cov_script
COPY run.sh ${WORKDIR}/run
COPY nginx.conf ${WORKDIR}
