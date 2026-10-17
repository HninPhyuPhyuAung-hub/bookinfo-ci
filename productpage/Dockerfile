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

FROM python:3.13.3-slim

# Pull in the latest Debian security patches (e.g. libgnutls30 CVE-2026-33845)
# on top of this pinned base image, since the base image tag itself isn't
# rebuilt every time Debian ships a security update.
#hadolint ignore=DL3008
RUN apt-get update \
    && apt-get upgrade -y \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /

COPY requirements.txt ./
RUN pip3 install -vvv --require-hashes --no-cache-dir -r requirements.txt

COPY test-requirements.txt ./
RUN pip3 install --no-cache-dir --require-hashes -r test-requirements.txt

COPY productpage.py /opt/microservices/
COPY tests/unit/* /opt/microservices/
COPY templates /opt/microservices/templates
COPY static /opt/microservices/static
COPY requirements.txt /opt/microservices/

ARG flood_factor
ENV FLOOD_FACTOR=${flood_factor:-0}

EXPOSE 9080
WORKDIR /opt/microservices
RUN python -m unittest discover

CMD ["gunicorn", "-b", "[::]:9080", "productpage:app", "-w", "8", "--keep-alive", "2", "-k", "gevent"]

USER 1000
