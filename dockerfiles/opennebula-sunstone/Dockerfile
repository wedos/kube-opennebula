FROM ubuntu:18.04 as builder
ARG VERSION=5.8.5

# Install build dependencies
RUN apt-get update \
 && apt-get install -y pbuilder debhelper ubuntu-dev-tools bash-completion \
      bison default-jdk flex javahelper libxmlrpc3-client-java \
      libxmlrpc3-common-java libxml2-dev ruby scons dh-systemd \
      npm ruby-ronn rename

# Download sources
RUN wget -O - https://github.com/OpenNebula/one/archive/release-$VERSION.tar.gz | tar xzf - \
 && mv one-release-$VERSION opennebula-$VERSION
WORKDIR /opennebula-$VERSION

# Make docs
RUN cd share/man \
 && ./build.sh

# Make sunstone
RUN cd src/sunstone/public \
 && ./build.sh -d \
 && export PATH=$PATH:$PWD/node_modules/.bin \
 && ./build.sh \
 && rm -rf node_modules/

# Prepare packages
RUN tar -czf /opennebula-$VERSION.tar.gz /opennebula-$VERSION/
RUN wget -O - https://github.com/OpenNebula/packages/archive/release-$VERSION.tar.gz | tar xzf - \
 && mv packages-release-$VERSION /packages-$VERSION
WORKDIR /packages-$VERSION
RUN sed -i '/^debuild/,$ s/^/#/' ubuntu1804.sh \
 && ./ubuntu1804.sh /opennebula-$VERSION.tar.gz

# Build packages
WORKDIR /root/build-Ubuntu-18.04/opennebula-$VERSION
RUN apt-get install -y $(dpkg-checkbuilddeps 2>&1 | sed 's/.*Unmet build dependencies: //') \
 && echo build_opennebula.tar.gz >> debian/source/include-binaries \
 && echo xmlrpc-c.tar.gz >> debian/source/include-binaries \
 && dpkg-buildpackage -us -uc \
 && mkdir -p /packages \
 && mv /root/build-Ubuntu-18.04/*.deb /packages/


FROM ubuntu:18.04

# Install opennebula-gems
COPY --from=builder packages/opennebula-common_*.deb packages/ruby-opennebula_*_all.deb packages/opennebula-tools_*.deb /packages/
RUN apt-get update \
 && apt-get -y install /packages/*.deb \
 && echo '0\n' | /usr/share/one/install_gems --yes \
 && mkdir -p /var/log/one /var/lock/one \
 && apt-get -y clean \
 && chown oneadmin: /var/log/one /var/lock/one

# Install opennebula-sunstone
COPY --from=builder packages/opennebula-sunstone_*.deb /packages/
RUN apt-get -y update \
 && ln -s /bin/true /usr/local/bin/systemd-tmpfiles \
 && dpkg -i packages/opennebula-sunstone_*.deb || true \
 && apt-get -yf install \
 && dpkg -l opennebula-sunstone \
 && rm -f /usr/local/bin/systemd-tmpfiles \
 && apt-get -y clean

# Fix permissions
RUN chown -R oneadmin:oneadmin /etc/one

# Logging to /dev/stdout, do not daemonize novnc-server
RUN sed -i 's|^\(SUNSTONE_LOG *=\).*|\1 "/dev/stdout"|' /usr/lib/one/sunstone/sunstone-server.rb \
 && sed -i 's|^\(VNC_LOG *=\).*|\1 "/dev/stdout"|' /usr/bin/novnc-server \
 && sed -i '/cmd *=/a\        exec(cmd)' /usr/lib/one/ruby/OpenNebulaVNC.rb

USER oneadmin
ENTRYPOINT [ "/usr/bin/ruby", "/usr/lib/one/sunstone/sunstone-server.rb" ]
#ENTRYPOINT [ "/usr/bin/ruby", "/usr/bin/novnc-server", "start" ]