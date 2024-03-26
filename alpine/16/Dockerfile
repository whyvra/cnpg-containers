ARG PG_BASE_IMAGE=16.2-alpine3.19

FROM postgres:$PG_BASE_IMAGE AS build

ARG PG_AUDIT_VERSION=REL_16_STABLE \
    PG_FAILOVER_VERSION=v1.0.1 \
    PGVECTOR_VERSION=v0.6.0 \
    PG_BARMAN_VERSION=3.10.0

RUN apk add -U --no-cache -t .build-deps1 \
		$DOCKER_PG_LLVM_DEPS \
		git \
		build-base \
		openssl-dev \
		krb5-dev

# Build and install pgaudit
WORKDIR /pgaudit
RUN git clone https://github.com/pgaudit/pgaudit --branch $PG_AUDIT_VERSION . \
	&& make install USE_PGXS=1 PG_CONFIG=/usr/local/bin/pg_config

# Build and install pg_failover_slots
WORKDIR /pg_failover_slots
RUN git clone https://github.com/EnterpriseDB/pg_failover_slots --branch $PG_FAILOVER_VERSION . \
	&& make install

# Build and install pgvector
WORKDIR /pgvector
RUN git clone https://github.com/pgvector/pgvector --branch $PGVECTOR_VERSION . \
	&& make install OPTFLAGS=""

# Build and install barman
WORKDIR /barman
RUN apk add -U --no-cache -t .run-deps \
		python3 \
		rsync \
		py3-argcomplete \
		py3-dateutil \
		py3-psycopg2 \
		py3-boto3 \
	&& apk add -U --no-cache -t .build-deps2 \
		py3-gpep517 \
		py3-setuptools \
		py3-wheel

RUN addgroup -S barman \
	&& adduser -SD -h /var/lib/barman/ -s /sbin/nologin -G barman -g barman barman

RUN git clone https://github.com/EnterpriseDB/barman --branch release/$PG_BARMAN_VERSION . \
	&& gpep517 build-wheel \
		--wheel-dir .dist \
		--output-fd 3 3>&1 >&2 \
	&& python3 -m installer -d "/" .dist/*.whl \
	&& cp doc/barman.conf /etc

WORKDIR /
# Remove source files and build deps
RUN rm -rf /pgaudit /pg_failover_slots /pgvector /barman \
	&& apk del .build-deps1 .build-deps2

# Change the uid of postgres to 26
RUN apk add --no-cache shadow \
	&& usermod -u 26 postgres \
	&& apk del shadow

# Copy all into scratch image to reduce image size, this will not be necessary
# when removing/squashing some layers in the build stage.
FROM scratch

ARG PG_BASE_IMAGE

ENV PG_MAJOR=16 \
    PG_VERSION=16.2 \
    DOCKER_PG_LLVM_DEPS="llvm15-dev clang15" \
    PGDATA=/var/lib/postgresql/data

# Do not split the description, otherwise we will see a blank space in the labels
LABEL name="PostgreSQL Container Images" \
    vendor="The CloudNativePG Contributors" \
    version="${PG_VERSION}" \
    release="6" \
    summary="PostgreSQL Container images." \
    description="This Docker image contains PostgreSQL and Barman Cloud based on Postgres ${PG_BASE_IMAGE}."

LABEL org.opencontainers.image.description="This Docker image contains PostgreSQL and Barman Cloud based on Postgres ${PG_BASE_IMAGE}."

COPY --from=build / /

USER 26
