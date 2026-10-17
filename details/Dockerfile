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

FROM ruby:3.4.3-slim

# Pull in the latest Debian security patches (e.g. libgnutls30 CVE-2026-33845)
# on top of this pinned base image, since the base image tag itself isn't
# rebuilt every time Debian ships a security update.
#hadolint ignore=DL3008
RUN apt-get update \
    && apt-get upgrade -y \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /opt/microservices
COPY Gemfile Gemfile.lock /opt/microservices/
RUN bundle install
# ruby:3.4.3-slim bundles its own copies of net-imap (0.5.6, CVE-2026-42257)
# and zlib (3.2.1, CVE-2026-27820) as "default gems". bundle install above
# installs patched replacements, but the vulnerable original files are still
# physically present on disk and keep getting flagged by image scanners even
# though the app never uses them - remove them outright. (zlib's compiled
# native extension lives separately at .../x86_64-linux/zlib.so and keeps
# working fine without its gemspec/source files - verified locally.)
RUN rm -rf \
    /usr/local/lib/ruby/gems/3.4.0/specifications/net-imap-0.5.6.gemspec \
    /usr/local/lib/ruby/gems/3.4.0/gems/net-imap-0.5.6 \
    /usr/local/lib/ruby/gems/3.4.0/cache/net-imap-0.5.6.gem \
    /usr/local/lib/ruby/gems/3.4.0/specifications/default/zlib-3.2.1.gemspec \
    /usr/local/lib/ruby/gems/3.4.0/gems/zlib-3.2.1

COPY details.rb /opt/microservices/

ARG service_version
ENV SERVICE_VERSION=${service_version:-v1}
ARG enable_external_book_service
ENV ENABLE_EXTERNAL_BOOK_SERVICE=${enable_external_book_service:-false}

EXPOSE 9080

CMD ["ruby", "details.rb", "9080"]

USER 1000
