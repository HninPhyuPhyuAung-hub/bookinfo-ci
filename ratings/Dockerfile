# Copyright Istio Authors
#
#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.

FROM node:23.11-slim

#hadolint ignore=DL3008
RUN apt-get update \
    && apt-get upgrade -y \
    && apt-get install curl ca-certificates --no-install-recommends -y \
    && update-ca-certificates \
    && rm -rf /var/lib/apt/lists/*

ARG service_version
ENV SERVICE_VERSION=${service_version:-v1}

# node:23.11-slim bundles npm 10.9.2, whose own vendored copy of `tar`
# (6.2.1) has CVE-2026-59873 (critical). npm 11.21.1 is the latest release
# still compatible with Node 23 (npm 12+ requires Node >=26) and bundles a
# patched tar (verified locally: 7.5.22).
RUN npm install -g npm@11.21.1

COPY package.json package-lock.json /opt/microservices/
WORKDIR /opt/microservices
RUN npm ci
COPY ratings.js /opt/microservices/

EXPOSE 9080
CMD ["node", "/opt/microservices/ratings.js", "9080"]

USER 1000
