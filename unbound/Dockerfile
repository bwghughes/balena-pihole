FROM balenalib/raspberrypi3-alpine:latest-build AS build

ENV LIBRESSL_VERSION="2.8.3"
ENV LIBRESSL_SHA="3967e08b3dc2277bf77057ea1f11148df7f96a2203cd21cf841902f2a1ec11320384a001d01fa58154d35612f7981bf89d5b1a60a2387713d5657677f76cc682"

ENV UNBOUND_VERSION="1.9.0"
ENV UNBOUND_SHA="7dfa8e078507fc24a2d0938eea590389453bacfcac023f1a41af19350ea1f7b87d0c82d7eead121a11068921292a96865e177274ff27ed8b8868445f80f7baf6"

# install build dependencies
RUN install_packages build-base curl file linux-headers libevent libevent-dev expat expat-dev

# work in /tmp
WORKDIR /tmp/src/libressl

# download, verify, and extract libressl
RUN LIBRESSL_DOWNLOAD_URL="https://ftp.openbsd.org/pub/OpenBSD/LibreSSL/libressl-${LIBRESSL_VERSION}.tar.gz" \
	&& curl -sSL "${LIBRESSL_DOWNLOAD_URL}" -o libressl.tar.gz \
	&& echo "${LIBRESSL_SHA} *libressl.tar.gz" | sha512sum -c - \
	&& tar xzf libressl.tar.gz --strip-components=1 \
	&& rm -f libressl.tar.gz

# configure, compile, and install libressl
RUN CFLAGS="-DLIBRESSL_APPS=off -DLIBRESSL_TESTS=off" \
		./configure --prefix=/opt/libressl --enable-static=no \
	&& make -j$(getconf _NPROCESSORS_ONLN) \
	&& make install \
	&& rm -rf \
		/opt/libressl/share \
		/opt/libressl/include\* \
		/opt/libressl/lib/libtls.* \
		/opt/libressl/bin/ocspcheck \
		/opt/libressl/lib/pkgconfig \
		/opt/libressl/lib/*.la \
	&& strip --strip-all \
		/opt/libressl/bin/* \
		/opt/libressl/lib/lib*

# work in /tmp
WORKDIR /tmp/src/unbound

# download, verify, and extract unbound
RUN UNBOUND_DOWNLOAD_URL="https://www.unbound.net/downloads/unbound-${UNBOUND_VERSION}.tar.gz" \
	&& curl -sSL "${UNBOUND_DOWNLOAD_URL}" -o unbound.tar.gz \
	&& echo "${UNBOUND_SHA} *unbound.tar.gz" | sha512sum -c - \
	&& tar xzf unbound.tar.gz --strip-components=1 \
	&& rm -f unbound.tar.gz

# create unbound group and user
RUN addgroup -S -g 88 unbound \
	&& adduser -S -D -H -s /sbin/nologin -G unbound -g "Unbound DNS resolver" -u 88 unbound

# configure, compile, and install unbound
RUN RANLIB="gcc-ranlib" \
		./configure --prefix=/opt/unbound --with-pthreads \
		--with-username=unbound --with-ssl=/opt/libressl --with-libevent \
		--enable-event-api --enable-static=no --enable-pie  --enable-relro-now \
	&& make -j$(getconf _NPROCESSORS_ONLN) \
	&& make install \
	&& rm -rf \
		/opt/unbound/share \
		/opt/unbound/include \
		/opt/unbound/lib/pkgconfig \
		/opt/unbound/lib/*.la \
	&& strip --strip-all \
		/opt/unbound/lib/lib* \
		/opt/unbound/sbin/unbound \
		/opt/unbound/sbin/unbound-anchor \
		/opt/unbound/sbin/unbound-checkconf \
		/opt/unbound/sbin/unbound-control \
		/opt/unbound/sbin/unbound-host

# ----------------------------------------------------------------------------

FROM balenalib/raspberrypi3-alpine:latest-run

# install runtime dependencies
RUN install_packages libevent expat

# create unbound group and user
RUN addgroup -S -g 88 unbound \
	&& adduser -S -D -H -s /sbin/nologin -G unbound -g "Unbound DNS resolver" -u 88 unbound

# copy libressl and unbound from build
COPY --from=build /opt/ /opt/

# install default config file
WORKDIR /opt/unbound/etc/unbound
COPY unbound.conf .

# run start script on boot
COPY start.sh /
CMD ["/bin/bash", "/start.sh"]

# expose dns ports
EXPOSE 53/tcp 53/udp
