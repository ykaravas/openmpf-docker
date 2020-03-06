# syntax=docker/dockerfile:experimental

#############################################################################
# NOTICE                                                                    #
#                                                                           #
# This software (or technical data) was produced for the U.S. Government    #
# under contract, and is subject to the Rights in Data-General Clause       #
# 52.227-14, Alt. IV (DEC 2007).                                            #
#                                                                           #
# Copyright 2019 The MITRE Corporation. All Rights Reserved.                #
#############################################################################

#############################################################################
# Copyright 2019 The MITRE Corporation                                      #
#                                                                           #
# Licensed under the Apache License, Version 2.0 (the "License");           #
# you may not use this file except in compliance with the License.          #
# You may obtain a copy of the License at                                   #
#                                                                           #
#    http://www.apache.org/licenses/LICENSE-2.0                             #
#                                                                           #
# Unless required by applicable law or agreed to in writing, software       #
# distributed under the License is distributed on an "AS IS" BASIS,         #
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.  #
# See the License for the specific language governing permissions and       #
# limitations under the License.                                            #
#############################################################################


ARG BUILD_REGISTRY
ARG BUILD_TAG=latest
FROM ${BUILD_REGISTRY}openmpf_build:${BUILD_TAG} as openmpf_build


FROM centos:7

SHELL ["/bin/bash", "-o", "errexit", "-o", "pipefail", "-c"]

RUN yum update --assumeyes; \
    yum install --assumeyes https://download1.rpmfusion.org/free/el/rpmfusion-free-release-7.noarch.rpm; \
    yum install --assumeyes ffmpeg; \
    yum clean all; \
    rm --recursive /var/cache/yum/*;

# Use mount and cp shell command because the docker COPY command resolves symlinks which then causes duplicate copies
# of the libraries to be added to the image.
RUN --mount=type=bind,from=openmpf_build,source=/build-artifacts/install/lib,target=/tmp/mpf-libs \
    cd /tmp/mpf-libs; \
    # component executor binary's dependencies
    cp --preserve=links --no-dereference liblog4cxx* libactivemq-cpp* libQtCore* libapr* libXext* \
        /usr/lib64;

ENV MPF_HOME /opt/mpf

# Component executor binary
COPY --from=openmpf_build /build-artifacts/install/bin/amq_detection_component $MPF_HOME/bin/amq_detection_component

# Component executor log config
COPY --from=openmpf_build /build-artifacts/install/config/Log4cxxConfig.xml $MPF_HOME/config/


# print()'ed messsages from docker-entrypoint.py were not being shown at the right time.
# Only applies to stdin, stdout and stderr
ENV PYTHONUNBUFFERED 1

ENV MPF_LOG_PATH $MPF_HOME/share/logs

ENV COMPONENT_VIRTUALENV $MPF_HOME/plugin-venv

ENV PLUGINS_DIR $MPF_HOME/plugins

ENV PATH $COMPONENT_VIRTUALENV/bin:/scripts:$PATH

COPY docker-executor-entrypoint.py /scripts/docker-entrypoint.py

ENTRYPOINT ["python", "/scripts/docker-entrypoint.py"]


LABEL org.label-schema.license="Apache 2.0" \
      org.label-schema.name="OpenMPF Python Component Executor" \
      org.label-schema.schema-version="1.0" \
      org.label-schema.url="https://openmpf.github.io" \
      org.label-schema.vcs-url="https://github.com/openmpf/openmpf-python-component-sdk" \
      org.label-schema.vendor="MITRE"