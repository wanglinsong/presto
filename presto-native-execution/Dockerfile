# prestissimo

# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

#============================================================
# /////////////////////////////////
#    BUILD TASK DEFINITION
# /////////////////////////////////
#============================================================
FROM quay.io/centos/centos:stream8 AS Builder

ARG CPU_TARGET=avx

USER root:root

ENV CPU_TARGET=${CPU_TARGET}

ENV CC=/opt/rh/gcc-toolset-9/root/bin/gcc
ENV CXX=/opt/rh/gcc-toolset-9/root/bin/g++

ENV PRESTO_HOME=/opt/presto
ENV DEPENDENCY_DIR=/opt/dependency
ENV INSTALL_PREFIX=/usr/local
ENV CMAKE_PREFIX_PATH=/usr/local
ENV PROMPT_ALWAYS_RESPOND=n

ENV PATH=${DEPENDENCY_DIR}/install/bin:${DEPENDENCY_DIR}/install/bin/hadoop-2.10.1/bin:$PATH

ENV PRESTO_ENABLE_PARQUET=ON
ENV PRESTO_ENABLE_S3=ON
ENV PRESTO_ENABLE_HDFS=ON
ENV PRESTO_ENABLE_TESTING=ON
ENV CLEANUP_BUILD_ARTIFACTS=1

SHELL ["/bin/bash", "-c"]
#-----------------------------------------------------------
#---( START of PrestoDB and Velox dependency tree build scripts
#-----------------------------------------------------------
WORKDIR /opt/presto
COPY . .
RUN set -exu && \
    dnf -y install openssh-clients git wget unzip make && \
    wget https://github.com/ninja-build/ninja/releases/download/v1.11.1/ninja-linux.zip && \
    unzip ninja-linux.zip -d /usr/local/bin && \
    mkdir -p "${DEPENDENCY_DIR}/install/bin" \
             "./_build/release" \
             "./catalog" \
             "./tests" && \
    dnf clean all

RUN set -exu && \
    bash "./scripts/setup-centos.sh" && \
    python3 -m pip install six && \
    set +exu && \
    source /opt/rh/gcc-toolset-9/enable && \
    set -exu && \
#-----------------------------------------------------------
#---( START wget valid antlr java runtime
#-----------------------------------------------------------
    wget -q 'https://www.antlr.org/download/antlr-4.9.3-complete.jar' -O '/usr/local/lib/antlr-4.9.3-complete.jar' && \
    printf '#!/usr/bin/bash\n\njava -jar /usr/local/lib/antlr-4.9.3-complete.jar "$@"' > '/usr/local/bin/antlr' && \
    chmod +x "/usr/local/bin/antlr" && \
    bash "./velox/scripts/setup-adapters.sh" && \
    dnf clean all

#-----------------------------------------------------------
#---( START of PrestoCpp and Velox main build process
#-----------------------------------------------------------
RUN source /opt/rh/gcc-toolset-9/enable && \
    set -exu && \
    make release \
        PRESTO_ENABLE_PARQUET=${PRESTO_ENABLE_PARQUET} \
        PRESTO_ENABLE_S3=${PRESTO_ENABLE_S3} \
        PRESTO_ENABLE_HDFS=${PRESTO_ENABLE_HDFS} \
        PRESTO_ENABLE_TESTING=${PRESTO_ENABLE_TESTING} \
        TREAT_WARNINGS_AS_ERRORS=OFF


#============================================================
#---) END of PrestoCpp and Velox main build process
#============================================================
# /////////////////////////////////
#    RUNTIME TASK DEFINITION
# /////////////////////////////////
#============================================================
FROM quay.io/centos/centos:stream8

LABEL org.opencontainers.image.title="Prestissimo" \
      org.opencontainers.image.description="Prestissimo Runtime Docker image" \
      org.opencontainers.image.version="1.0.0"

ARG PRESTO_UID=186

SHELL ["/bin/bash", "-c"]

ENV CPU_TARGET=${CPU_TARGET}
ENV TZ=Europe/Warsaw

WORKDIR /opt/presto/

COPY --chown=${PRESTO_UID} scripts/release-centos-dockerfile/opt/ /opt/
COPY --chown=${PRESTO_UID} --from=Builder /opt/presto/ /opt/presto/

RUN set -exu && \
    dnf -y install uuid hostname vim git && \
    #---( INFO create a system account (-r) and same name group (-U) with UID and home path at /opt (-d) named presto
    useradd -r -U --uid=${PRESTO_UID} -d /opt presto && \
    chgrp ${PRESTO_UID} /etc/passwd && \
    chown -R ${PRESTO_UID} /opt && \
    chmod -R ug+rw /etc/passwd /opt && \
    chmod ug+x /opt/*.sh /opt/presto/presto_server && \
    rm -rf /root/.cache /tmp && \
    dnf clean all

USER ${PRESTO_UID}

COPY --chown=${PRESTO_UID} scripts/release-centos-dockerfile/etc/ /opt/presto/etc/
COPY --chown=${PRESTO_UID} --from=Builder /usr/local/lib64/lib* /usr/lib64/lib* /usr/local/lib64/
COPY --chown=${PRESTO_UID} --from=Builder /usr/local/lib/lib* /usr/local/lib/

ENTRYPOINT [ "/opt/entrypoint.sh" ]

EXPOSE 22 8080

#============================================================
# /////////////////////////////////
#  END OF RUNTIME TASK DEFINITION
# /////////////////////////////////
#============================================================
