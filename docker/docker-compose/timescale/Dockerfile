# PostgreSQL with pgvector, pgvectorscale, and pg_textsearch extensions
# All three extensions from Timescale/pgvector for high-performance vector and text search
# Note: Requires PostgreSQL 16+
# Multi-stage build: the builder stage compiles everything, the runtime
# stage copies only the installed extension files into a clean postgres
# image, so the Rust toolchain / LLVM / dev headers never ship (they are
# what made the single-stage image ~3.5 GB).
FROM postgres:18 AS builder

# Install build dependencies and Rust toolchain
RUN apt-get update && apt-get install -y \
    build-essential \
    git \
    postgresql-server-dev-18 \
    libpq-dev \
    cmake \
    curl \
    pkg-config \
    libssl-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Rust toolchain (required for pgvectorscale)
# Retry: static.rust-lang.org drops connections often enough on slow links
# that a single failed fetch shouldn't waste the whole (long) build.
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y \
    || (sleep 5 && curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y) \
    || (sleep 15 && curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y)
ENV PATH="/root/.cargo/bin:${PATH}"

# Install pgvector (required by pgvectorscale). v0.8.0 predates Postgres 18's
# vacuum_delay_point(bool) signature; 0.8.1+ is needed for the PG18 headers.
RUN cd /tmp && \
    git clone --branch v0.8.7 https://github.com/pgvector/pgvector.git && \
    cd pgvector && \
    make && \
    make install && \
    rm -rf /tmp/pgvector

# Install cargo-pgrx (PostgreSQL extension framework for Rust)
# pgvectorscale 0.9.1 pins pgrx exactly; keep the two in sync. 0.9.1 also
# raises DiskANN's indexed-dimension ceiling from 2000 to 16000 (SBQ-quantized
# above 2000), which older cargo-pgrx/pgvectorscale pairs could not index.
RUN cargo install cargo-pgrx --version 0.16.1 --locked && \
    cargo pgrx init --pg18 /usr/bin/pg_config

# Install pgvectorscale (DiskANN index support)
RUN cd /tmp && \
    git clone --branch 0.9.1 https://github.com/timescale/pgvectorscale.git && \
    cd pgvectorscale/pgvectorscale && \
    cargo pgrx install --release && \
    rm -rf /tmp/pgvectorscale

# Install pg_textsearch (BM25 text search)
RUN cd /tmp && \
    git clone https://github.com/timescale/pg_textsearch.git && \
    cd pg_textsearch && \
    make && \
    make install && \
    rm -rf /tmp/pg_textsearch

FROM postgres:18 AS runtime

# Installed extensions land in exactly these two trees (the builder and this
# stage share the same postgres:18 base, so the copy only adds our files).
COPY --from=builder /usr/lib/postgresql/18/lib/ /usr/lib/postgresql/18/lib/
COPY --from=builder /usr/share/postgresql/18/extension/ /usr/share/postgresql/18/extension/

# Ensure extensions are preloaded (pg_textsearch requires preloading)
RUN echo "shared_preload_libraries = 'pg_textsearch'" >> /usr/share/postgresql/postgresql.conf.sample
