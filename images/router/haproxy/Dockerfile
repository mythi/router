FROM clearlinux:base as builder

ARG QAT_DRIVER_RELEASE="qat1.7.l.4.9.0-00008"
ARG QAT_ENGINE_VERSION="v0.5.44"
ARG QAT_ENGINE_FLAGS="--disable-multibuff-offload"

RUN swupd bundle-add --skip-diskspace-check devpkg-systemd devpkg-openssl c-basic wget git go-basic && \
    git clone -b $QAT_ENGINE_VERSION https://github.com/intel/QAT_Engine && \
    git clone https://github.com/intel/ipp-crypto && \
    git clone --depth 1 -b release-4.4 https://github.com/openshift/router /router && \
    wget https://01.org/sites/default/files/downloads/$QAT_DRIVER_RELEASE.tar.gz && \
    tar xf *.tar.gz

RUN sed -i -e 's/cmn_ko$//' -e 's/lac_kernel$//' quickassist/Makefile && \
    KERNEL_SOURCE_ROOT=/tmp ./configure --enable-icp-debug --enable-icp-trace && \
    make quickassist-all adf-ctl-all && \
    install -m 755 build/libqat_s.so /usr/lib/ && \
    install -m 755 build/libusdm_drv_s.so /usr/lib/ && \
    install -m 755 build/adf_ctl /usr/bin/ && \
    cd /ipp-crypto/sources/ippcp/crypto_mb && \
    sed -i -e '/OPENSSL_USE_STATIC_LIBS/d' CMakeLists.txt && \
    cmake . -B"../build" \
    -DOPENSSL_INCLUDE_DIR=/usr/include/openssl \
    -DOPENSSL_LIBRARIES=/usr/lib64 \
    -DOPENSSL_ROOT_DIR=/usr/bin/openssl && \
    cd ../build && \
    make crypto_mb && make install && \
    install -m 755 bin/vfy_ifma_cp_rsa_mb /usr/bin/ && \
    install -m 755 bin/vfy_ifma_rsa_mb /usr/bin/ && \
    cd /QAT_Engine && \
    ./autogen.sh && \
    ./configure \
    $QAT_ENGINE_FLAGS \
    --with-qat_dir=/ \
    --enable-qat_debug \
    --disable-qat_ecx \
    --with-openssl_install_dir=/usr/lib64 \
    --with-qat_install_dir=/usr/lib \
    --with-multibuff_dir=/ipp-crypto/sources/ippcp/crypto_mb \
    --with-multibuff_install_dir=/usr/local/lib \
    --enable-openssl_install_build_arch_path && \
    make && make install && \
    cd /router && make

FROM clearlinux:base

RUN INSTALL_PKGS="haproxy rsyslog curl strace" && \
    swupd bundle-add $INSTALL_PKGS && \
    mkdir -p /var/lib/haproxy/router/{certs,cacerts,whitelists} && \
    mkdir -p /var/lib/haproxy/{conf/.tmp,run,bin,log} && \
    touch /var/lib/haproxy/conf/{{os_http_be,os_edge_reencrypt_be,os_tcp_be,os_sni_passthrough,os_route_http_redirect,cert_config,os_wildcard_domain}.map,haproxy.config} && \
    setcap 'cap_net_bind_service=ep' /usr/sbin/haproxy && \
    chown -R :0 /var/lib/haproxy && \
    chmod -R g+w /var/lib/haproxy
COPY --from=builder /router/openshift-router /usr/bin/
COPY --from=builder /usr/lib/libqat_s.so /usr/lib/
COPY --from=builder /usr/lib/libusdm_drv_s.so /usr/lib/
COPY --from=builder /usr/bin/adf_ctl /usr/bin
COPY --from=builder /usr/bin/vfy_ifma_cp_rsa_mb /usr/bin
COPY --from=builder /usr/bin/vfy_ifma_rsa_mb /usr/bin
COPY --from=builder /usr/lib64/engines-1.1/qat.so /usr/lib64/engines-1.1/qat.so
COPY --from=builder /LICENSE.GPL /usr/share/package-licenses/libqat/LICENSE.GPL
COPY --from=builder /QAT_Engine/LICENSE /usr/share/package-licenses/QAT_Engine/LICENSE
COPY --from=builder /ipp-crypto/LICENSE /usr/share/package-licenses/ipp-crypto/LICENSE
COPY images/router/haproxy/ /var/lib/haproxy/
LABEL io.k8s.display-name="OpenShift HAProxy Router" \
      io.k8s.description="This component offers ingress to an OpenShift cluster via Ingress and Route rules." \
      io.openshift.tags="openshift,router,haproxy"
EXPOSE 80 443
WORKDIR /var/lib/haproxy/conf
ENV TEMPLATE_FILE=/var/lib/haproxy/conf/haproxy-config.template \
    RELOAD_SCRIPT=/var/lib/haproxy/reload-haproxy
ENTRYPOINT ["/usr/bin/openshift-router"]
