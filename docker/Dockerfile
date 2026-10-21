FROM ubuntu:24.04 AS runner-builder

# gss-labo: 2.336.0 は Sdk.csproj の System.Security.Cryptography.Pkcs 10.0.7 に脆弱性勧告（GHSA-2fmv-rv2c-59x7 /
# GHSA-846f-f764-76mx）が出て NuGet restore が NU1902/NU1903（警告をエラー扱い）で落ち、再ビルドできなくなった
# （2026-10-21）。Pkcs 10.0.12 の 2.338.0 に上げる。NuGetAudit を切って古い版のまま通すことはしない。
ARG RUNNER_VERSION=2.338.0
ARG RUNNER_SOURCE_REVISION=197ca027b90de199464dee3bb9ff198a1db60967

RUN apt-get update \
  && apt-get install --yes --no-install-recommends \
    ca-certificates \
    curl \
    git \
    libicu74 \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /opt/actions-runner-source
RUN git clone --depth 1 --branch "v${RUNNER_VERSION}" https://github.com/actions/runner.git . \
  && test "$(git rev-parse HEAD)" = "${RUNNER_SOURCE_REVISION}"
# gss-labo: 上流 actions/runner#4777（merge d1d30d1f1c3c4e21cd4fadd69f6683bae1eb6084）の先取り。
# 2.338.0 の Pkcs / Asn1 10.0.12 も GHSA-2fmv-rv2c-59x7 / GHSA-846f-f764-76mx（2026-10-13 公開、修正版 10.0.13）の
# 対象で NuGet restore が落ちるため、src/Sdk/Sdk.csproj のこの 2 パッケージの版だけを 10.0.13 に上げる。
# 置換前の 10.0.12 が 2 件見つからない・置換後の 10.0.13 が 2 件でない場合はビルドを落とす。
# actions runner を 2.339.0 以降（#4777 を含むリリース）に上げたらこの段を外す。
RUN set -eu \
  && csproj=src/Sdk/Sdk.csproj \
  && pinned='<PackageReference Include="System\.(Security\.Cryptography\.Pkcs|Formats\.Asn1)" Version="' \
  && test "$(grep -cE "${pinned}10\.0\.12\" />" "$csproj")" = 2 \
  && sed -i -E "s#(${pinned})10\.0\.12(\" />)#\\110.0.13\\3#" "$csproj" \
  && test "$(grep -cE "${pinned}10\.0\.13\" />" "$csproj")" = 2 \
  && test "$(grep -cE 'Include="System\.(Security\.Cryptography\.Pkcs|Formats\.Asn1)"' "$csproj")" = 2
COPY docker/actions-runner-results-proxy.patch /tmp/actions-runner-results-proxy.patch
RUN git apply /tmp/actions-runner-results-proxy.patch \
  && cd src \
  && ./dev.sh layout Release

# The runner invokes node20 to evaluate workflow expressions (for example,
# hashFiles()) and can still execute Node 20 actions during GitHub's temporary
# Node 24 opt-out. This runner cannot execute Docker container actions, so only
# the Node copies used inside Alpine container steps are unreachable. Prune
# those in the builder stage so they are absent from the final image layer.
RUN test -x /opt/actions-runner-source/_layout/externals/node20/bin/node \
  && test -x /opt/actions-runner-source/_layout/externals/node24/bin/node \
  && rm -rf \
    /opt/actions-runner-source/_layout/externals/node20_alpine \
    /opt/actions-runner-source/_layout/externals/node24_alpine

FROM ubuntu:24.04

ARG RUNNER_VERSION=2.338.0
ARG NODE_VERSION=22.23.2
ARG NODE_SHA256=d60acfe00a2932254bb0ad20e01b0d74397a0875595de719654b214f4b03f307

ENV DEBIAN_FRONTEND=noninteractive \
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true \
  RUNNER_TOOL_CACHE=/opt/hostedtoolcache \
  AGENT_TOOLSDIRECTORY=/opt/hostedtoolcache \
  CF_RESOURCE_TRACE_PATH=/home/runner/.cf-runner/resource-trace.csv \
  CF_RESOURCE_MARKERS_PATH=/home/runner/.cf-runner/resource-markers.tsv \
  ImageOS=cloudflare-ubuntu-24.04 \
  ImageVersion=cloudflare-runner-${RUNNER_VERSION}

RUN apt-get update \
  && apt-get install --yes --no-install-recommends \
    build-essential \
    bzip2 \
    ca-certificates \
    cmake \
    curl \
    jq \
    ninja-build \
    openssh-client \
    pkg-config \
    python3 \
    rsync \
    software-properties-common \
    tini \
    unzip \
    wget \
    xz-utils \
    zip \
    zstd \
  && add-apt-repository --yes ppa:git-core/ppa \
  && apt-get update \
  && apt-get install --yes --no-install-recommends git git-lfs \
  && useradd --create-home --shell /bin/bash runner \
  && mkdir --parents /home/runner/actions-runner \
  && mkdir --parents "${RUNNER_TOOL_CACHE}/node/${NODE_VERSION}" \
  && curl --fail --location --retry 3 \
    --output /tmp/node.tar.xz \
    "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-x64.tar.xz" \
  && echo "${NODE_SHA256}  /tmp/node.tar.xz" | sha256sum --check --status \
  && tar --extract --xz --file /tmp/node.tar.xz --directory /usr/local --strip-components=1 \
  && rm /tmp/node.tar.xz \
  && ln --symbolic /usr/local "${RUNNER_TOOL_CACHE}/node/${NODE_VERSION}/x64" \
  && touch "${RUNNER_TOOL_CACHE}/node/${NODE_VERSION}/x64.complete" \
  && corepack enable \
  && rm -rf /var/lib/apt/lists/*

COPY --from=runner-builder --chown=runner:runner /opt/actions-runner-source/_layout/ /home/runner/actions-runner/

RUN /home/runner/actions-runner/bin/installdependencies.sh \
  && rm -rf /var/lib/apt/lists/* \
  && chown runner:runner /home/runner/actions-runner \
  && chown --recursive runner:runner "${RUNNER_TOOL_CACHE}"

# PostgreSQL 17（PGDG）・redis-server・gh CLI（gss-labo 追加、2026-10-21）
# 目的: `services: postgres:17`・`sudo apt-get`・gh に依存して GitHub-hosted に残っている job
# （realdb-migration / talk-backend / workflow-gate）を CF runner へ移せるようにする。
# - sudo は入れない。apt ではサービスを起動・クラスタ作成させず（policy-rc.d と
#   create_main_cluster = false）、job が runner ユーザーで $RUNNER_TEMP 配下に起動する。
# - initdb / pg_ctl / postgres 等サーバー側のバイナリは /usr/lib/postgresql/17/bin。
#   PATH の末尾に足す（psql / createdb / dropdb は /usr/bin の pg_wrapper でも 17 が選ばれる）。
#   起動例（手順と検査の正本は .github/scripts/cf-runner-native-services-check.sh）:
#     initdb -D "$RUNNER_TEMP/pgdata" -U postgres -E UTF8 --no-locale -A trust \
#       -c listen_addresses=127.0.0.1 -c port=5432 \
#       -c unix_socket_directories="$RUNNER_TEMP/pgsock" -c dynamic_shared_memory_type=mmap
#     pg_ctl -D "$RUNNER_TEMP/pgdata" -l "$RUNNER_TEMP/postgres.log" -w start
#     redis-server --bind 127.0.0.1 --port <一時 port> --save '' --appendonly no --daemonize yes
# - 署名鍵は主鍵の指紋を照合してから信頼する。GitHub CLI の keyring は旧鍵（2026-09-05 失効）と
#   現行の署名鍵の 2 本入り。どちらかの鍵が差し替わったらビルドが落ちるので、指紋を確かめて更新する。
# - PostgreSQL はメジャー 17 に固定（CI の `services: postgres:17` と揃える。マイナーは PGDG の最新）。
ARG POSTGRES_MAJOR=17
ARG PGDG_SIGNING_FPR=B97B0AFCAA1A47F044F244A07FCC7D46ACCC4CF8
ARG GH_CLI_SIGNING_FPRS="2C6106201985B60E6C7AC87323F3D4EA75716059 7F38BBB59D064DBCB3D84D725612B36462313325"
RUN set -eu \
  && created_policy_rc_d=0 \
  && if [ ! -e /usr/sbin/policy-rc.d ]; then \
    printf '#!/bin/sh\nexit 101\n' > /usr/sbin/policy-rc.d \
    && chmod 0755 /usr/sbin/policy-rc.d \
    && created_policy_rc_d=1; \
  fi \
  && apt-get update \
  && apt-get install --yes --no-install-recommends gpg \
  && install --directory --mode 0755 /etc/apt/keyrings \
  && curl --fail --location --retry 3 \
    --output /etc/apt/keyrings/postgresql-pgdg.asc \
    https://www.postgresql.org/media/keys/ACCC4CF8.asc \
  && curl --fail --location --retry 3 \
    --output /etc/apt/keyrings/githubcli-archive-keyring.gpg \
    https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  && export GNUPGHOME="$(mktemp -d)" \
  && primary_fingerprints() { \
    gpg --show-keys --with-colons "$1" \
      | awk -F: '$1 == "pub" { want = 1; next } want && $1 == "fpr" { print $10; want = 0 }' \
      | sort | tr '\n' ' ' | sed 's/ $//'; \
  } \
  && test "$(primary_fingerprints /etc/apt/keyrings/postgresql-pgdg.asc)" = "${PGDG_SIGNING_FPR}" \
  && test "$(primary_fingerprints /etc/apt/keyrings/githubcli-archive-keyring.gpg)" = "${GH_CLI_SIGNING_FPRS}" \
  && rm -rf "$GNUPGHOME" \
  && unset GNUPGHOME \
  && codename="$(. /etc/os-release && echo "$VERSION_CODENAME")" \
  && printf 'Types: deb\nURIs: https://apt.postgresql.org/pub/repos/apt\nSuites: %s-pgdg\nComponents: main\nSigned-By: /etc/apt/keyrings/postgresql-pgdg.asc\n' \
    "$codename" > /etc/apt/sources.list.d/pgdg.sources \
  && printf 'Types: deb\nURIs: https://cli.github.com/packages\nSuites: stable\nComponents: main\nArchitectures: %s\nSigned-By: /etc/apt/keyrings/githubcli-archive-keyring.gpg\n' \
    "$(dpkg --print-architecture)" > /etc/apt/sources.list.d/github-cli.sources \
  && apt-get update \
  && apt-get install --yes --no-install-recommends postgresql-common \
  && sed -ri 's/^#?[[:space:]]*create_main_cluster[[:space:]]*=.*/create_main_cluster = false/' \
    /etc/postgresql-common/createcluster.conf \
  && grep -qx 'create_main_cluster = false' /etc/postgresql-common/createcluster.conf \
  && apt-get install --yes --no-install-recommends \
    "postgresql-${POSTGRES_MAJOR}" \
    "postgresql-client-${POSTGRES_MAJOR}" \
    redis-server \
    gh \
  && test -x "/usr/lib/postgresql/${POSTGRES_MAJOR}/bin/initdb" \
  && test -x "/usr/lib/postgresql/${POSTGRES_MAJOR}/bin/pg_ctl" \
  && test ! -e "/var/lib/postgresql/${POSTGRES_MAJOR}/main" \
  && if [ "$created_policy_rc_d" = 1 ]; then rm /usr/sbin/policy-rc.d; fi \
  && rm -rf /var/lib/apt/lists/*

ENV PATH="${PATH}:/usr/lib/postgresql/${POSTGRES_MAJOR}/bin"

COPY --chown=runner:runner docker/start-runner.sh /home/runner/start-runner.sh
COPY --chown=runner:runner docker/invalid-runner-hook.sh /home/runner/invalid-runner-hook.sh
COPY --chown=runner:runner docker/job-started-hook.sh /home/runner/job-started-hook.sh
COPY docker/report-container-usage.sh /usr/local/bin/report-container-usage
COPY --chown=runner:runner docker/resource-trace.sh /usr/local/bin/resource-trace
COPY --chown=runner:runner docker/cf-resource-mark.sh /usr/local/bin/cf-resource-mark
COPY --chown=runner:runner docker/cf-resource-trace.sh /usr/local/bin/cf-resource-trace
COPY --chown=runner:runner docker/runner-results-proxy.mjs /usr/local/bin/runner-results-proxy
RUN chmod 0755 \
  /home/runner/start-runner.sh \
  /home/runner/invalid-runner-hook.sh \
  /home/runner/job-started-hook.sh \
  /usr/local/bin/report-container-usage \
  /usr/local/bin/resource-trace \
  /usr/local/bin/cf-resource-mark \
  /usr/local/bin/cf-resource-trace \
  /usr/local/bin/runner-results-proxy

USER runner
WORKDIR /home/runner/actions-runner

ENTRYPOINT ["/usr/bin/tini", "--", "/home/runner/start-runner.sh"]
