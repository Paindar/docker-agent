ARG JAVA_VERSION=17.0.8.1_1
FROM eclipse-temurin:"${JAVA_VERSION}"-jdk-jammy AS jre-build

# This Build ARG is populated by Docker
# Ref. https://docs.docker.com/engine/reference/builder/#automatic-platform-args-in-the-global-scope
ARG TARGETPLATFORM

SHELL ["/bin/bash","-e", "-u", "-o", "pipefail", "-c"]

# Generate smaller java runtime without unneeded files
# for now we include the full module path to maintain compatibility
# while still saving space (approx 200mb from the full distribution)
RUN if test "${TARGETPLATFORM}" != 'linux/arm/v7'; then \
    case "$(jlink --version 2>/dev/null)" in \
        # jlink version 11 has less features than JDK17+
        "11."*) strip_java_debug_flags=("--strip-debug") ;; \
        *) strip_java_debug_flags=("--strip-java-debug-attributes") ;; \
    esac; \
    if test "${TARGETPLATFORM}" != 'linux/amd/64'; then \
        # posix_spawn (default for JDK12+) fails erratically with qemu on non x86 CPUs - https://github.com/openzipkin/docker-java/issues/34#issuecomment-721673618
        export JAVA_TOOL_OPTIONS='-Djdk.lang.Process.launchMechanism=vfork'; \
    fi; \
    jlink \
        --add-modules ALL-MODULE-PATH \
        "${strip_java_debug_flags[@]}" \
        --no-man-pages \
        --no-header-files \
        --compress=2 \
        --output /javaruntime; \
    # It is acceptable to have a larger image in arm/v7 (arm 32 bits) environment.
    # Because jlink fails with the error "jmods: Value too large for defined data type" error.
    else cp -r /opt/java/openjdk /javaruntime; \
    fi

FROM centos:centos7.5.1804

ARG user=jenkins
ARG group=jenkins
ARG uid=1000
ARG gid=1000
ARG JENKINS_AGENT_HOME=/home/${user}

ENV JENKINS_AGENT_HOME=${JENKINS_AGENT_HOME}
ARG AGENT_WORKDIR="${JENKINS_AGENT_HOME}/agent"

ENV AGENT_WORKDIR=${AGENT_WORKDIR}

RUN groupadd -g ${gid} ${group} \
    && useradd -d "${JENKINS_AGENT_HOME}" -u "${uid}" -g "${gid}" -m -s /bin/bash "${user}" \
    # Prepare subdirectories
    && mkdir -p "${JENKINS_AGENT_HOME}/.ssh/" "${AGENT_WORKDIR}" "${JENKINS_AGENT_HOME}/.jenkins" \
    # Make sure that user 'jenkins' own these directories and their content
    && chown -R "${uid}":"${gid}" "${JENKINS_AGENT_HOME}" "${AGENT_WORKDIR}"

# Import svn repo
COPY subversion.repo /etc/yum.repos.d/subversion.repo

# Install necessary tools
RUN yum update -y && \
    yum install -y \
    ca-certificates \
    java-1.8.0-openjdk-devel \
    subversion \
    bzip2\
    python3\
    cmake\
    make\
    git \
    gcc \
    gcc-c++ \
    openssh-server \
    glibc-common

RUN pip3 install ply
RUN yum clean all

# Set ssh config.
RUN sed -i /etc/ssh/sshd_config \
        -e 's/#PermitRootLogin.*/PermitRootLogin no/' \
        -e 's/#PubkeyAuthentication.*/PubkeyAuthentication yes/'  \
        -e 's/#PasswordAuthentication.*/PasswordAuthentication no/' \
        -e 's/#SyslogFacility.*/SyslogFacility AUTH/' \
        -e 's/#LogLevel.*/LogLevel INFO/' && \
    mkdir /var/run/sshd

# VOLUME directive must happen after setting up permissions and content
VOLUME "${AGENT_WORKDIR}" "${JENKINS_AGENT_HOME}"/.jenkins "/tmp" "/run" "/var/run"
WORKDIR "${JENKINS_AGENT_HOME}"

# Generate C.UTF-8 locale
RUN localedef -i en_US -f UTF-8 en_US.UTF-8
# Set the locale
ENV LANG='en_US.UTF-8' LC_ALL='en_US.UTF-8'

ENV JAVA_HOME=/opt/java/openjdk
ENV PATH="${JAVA_HOME}/bin:${PATH}"
COPY --from=jre-build /javaruntime $JAVA_HOME

RUN echo "PATH=${PATH}" >> /etc/environment
COPY setup-sshd /usr/local/bin/setup-sshd
RUN chmod +x /usr/local/bin/setup-sshd

ENTRYPOINT ["setup-sshd"]