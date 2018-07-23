---
layout: post
title:  "Preparations for Baking"
date:   2018-07-23 07:24:29 +0000
author: xtzbaker
---

## Infrastructure Provider

Whilst it is possible to setup your nodes on a VPS/Virtual Machine, we recommend obtaining dedicated servers.  Cheap BareMetal providers are listed below, we recomend enabling MFA.

- Vultr
- Scaleway
- Kimsufi (ovh or soyoustart are the more premium versions of this provider)

## Compile Tezos Binaries

Many Tezos tutorials compile and run the Tezos node/client on the same machine.  This is fairly poor practice from a security perspective, your Tezos node will have a lot of software installed/running that is not required by the Tezos software at runtime.  In an effort to reduce the attack surface we recommend compiling the software on a clean box and copying only the binaries you need to your hardened Tezos nodes.

If you need assistance compiling Tezos we have found one of the best guides for this is provided by the [Tezos Community](https://github.com/tezoscommunity/FAQ/blob/master/Compile_Betanet.md)

In addition we maintain compiled versions of Tezos in our [Github repository](https://github.com/xtzbaker/tezos/releases), however we always recommend you compile it yourself and not trust a third party
