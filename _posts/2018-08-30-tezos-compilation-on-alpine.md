---
layout: post
title:  "Compiling Tezos on Alpine for Raspberry Pi 3"
date:   2018-08-30 07:24:29 +0000
author: xtzbaker
---

We wanted to get our baking underway before we had fully implemented our cloud infrastructure so we decided to hook up a Raspberry Pi to our public nodes and bake from our Ledger Nano S. 

Whilst we were waiting for the Pi to be delivered we wanted to make sure we could get things up and running as we wanted.  Again we were aiming for a minimal installation of linux with just what we need for baking.  Thankfully [Alpine Linux has just released](https://alpinelinux.org/posts/Alpine-3.8.0-released.html) a version that supports the Raspberry Pi 3 Model B+.  For those unfamiliar with Alpine Linux it is a distribution with a focus on security and having a small footprint, this is perfect for our needs.

With our distro chosen we then had to make sure we are comfortable getting Tezos running under Alpine. Unfortunately many of the provided tools and instructions don't work for Alpine so we experimented with Docker and ended up with the following multi stage Dockerfile:

```
FROM alpine:3.8 as opam-builder

RUN apk update && apk upgrade && \
    apk add build-base bzip2 git tar curl ca-certificates patch openssl && \
    wget https://github.com/ocaml/opam/archive/2.0.0-rc4.tar.gz && \
    tar xzf 2.0.0-rc4.tar.gz && mv opam-2.0.0-rc4 /tmp/opam && \
    cd /tmp/opam && \
    cd /tmp/opam && \
    make cold && \
    mkdir -p /usr/local/bin && \
    cp /tmp/opam/opam /usr/local/bin/opam && \
    cp /tmp/opam/opam-installer /usr/local/bin/opam-installer && \
    chmod a+x /usr/local/bin/opam /usr/local/bin/opam-installer && \
    # mkdir -p /usr/local/share/opam && \
    # cp shell/wrap-build.sh /usr/local/share/opam && \
    # echo 'wrap-build-commands: \"/usr/local/share/opam/wrap-build.sh\"' >> /etc/opamrc.userns && \
    # cp shell/wrap-install.sh /usr/local/share/opam && echo 'wrap-install-commands: \"/usr/local/share/opam/wrap-install.sh\"' >> /etc/opamrc.userns && \
    # cp shell/wrap-remove.sh /usr/local/share/opam && echo 'wrap-remove-commands: \"/usr/local/share/opam/wrap-remove.sh\"' >> /etc/opamrc.userns && \
    rm -rf /tmp/opam && \
    strip /usr/local/bin/opam*

FROM alpine:3.8 as tezos-builder

COPY --from=opam-builder /usr/local/bin/opam /usr/local/bin/opam

RUN echo "@testing http://nl.alpinelinux.org/alpine/edge/testing" >> /etc/apk/repositories && \
    apk update && apk upgrade && \
    apk add patch unzip make gcc m4 git g++ aspcud bubblewrap curl bzip2 rsync libev-dev gmp-dev pkgconf perl bash hidapi-dev@testing && \
    adduser -D tezos

USER tezos

ENV OPAMYES=true

RUN opam init  --compiler=4.06.1 --disable-sandboxing && \
    eval $(opam env) && \
    git clone -b betanet https://gitlab.com/tezos/tezos.git /tmp/tezos

WORKDIR /tmp/tezos

RUN eval $(opam env) && \
    make build-deps

RUN eval $(opam env) && \
    make

RUN strip /tmp/tezos/tezos-*

FROM alpine:3.8

COPY --from=tezos-builder /tmp/tezos/tezos-* /usr/local/bin/

RUN echo "@testing http://nl.alpinelinux.org/alpine/edge/testing" >> /etc/apk/repositories && \
    apk update && apk upgrade && \
    apk add gmp libev hidapi@testing && \
    mv -v /usr/local/bin/tezos-accuser* /usr/local/bin/tezos-accuser && \
    mv -v /usr/local/bin/tezos-baker* /usr/local/bin/tezos-baker && \
    mv -v /usr/local/bin/tezos-endorser* /usr/local/bin/tezos-endorser && \
    mv -v /usr/local/bin/tezos-admin-client* /usr/local/bin/tezos-admin-client && \
    mv -v /usr/local/bin/tezos-client* /usr/local/bin/tezos-client && \
    mv -v /usr/local/bin/tezos-node* /usr/local/bin/tezos-node && \
    mv -v /usr/local/bin/tezos-signer* /usr/local/bin/tezos-signer && \
    mv -v /usr/local/bin/tezos-protocol-compiler* /usr/local/bin/tezos-protocol-compiler 

CMD /bin/sh
```

There are 3 stages to the build:

1. Compile `opam` for Alpine.  None of the available builds on the opam repository seemed to work, no harm in being able to compile it ourselves anyway.
2. Compile `tezos` binaries.
3. The last stage contains a minimal image containing the tezos binaries and only the necessary runtime dependencies.  It's nice to see we only need to add 3 additional packages to get things running. 

With this exercise complete we're now comfortable that when our Raspberry Pi arrives we'll be in a position to get it up and running quickly with our Ledger Nano S and public nodes.

