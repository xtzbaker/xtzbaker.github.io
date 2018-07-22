---
layout: post
title:  "Secure Tezos Baking"
date:   2018-07-22 07:24:29 +0000
author: xtzbaker
---

This is the first post in a multi part tutorial, after which you will have a reasonably secure baking infrastructure on which you can bake Tezos and act as a delegation service if you chose.  Here we will outline the rough structure of our target architecture and follow up with subsequent posts implementing each part.

Whilst you are free to make use of any OS you like whilst following these guides, we will target Ubuntu Xenial (64-bit) so if you are using any other OS/Processor you will need to make the necessary amendments.

The main focus of these tutorials is to ensure the baking setup is secure, we can never guarantee security, but we aim to provide defense in depth whilst also minimizing the attack surface.  Acknowledging that we can't achieve 100% security, we recognise that the following are the primary attack points in the proposed infrastucture state:

- Access to the account for managing your infrastructure
- Vulnerabilities in Tezos software and the the Tezos protocol or smart contracts
- Key and Password management outside the deployed infrastructure

It is left to the reader to ensure they are taking appropriate measures to secure against attacks to the above.

With that out of the way let us describe what we intend to build.

1. A Private baking node - not accessible by the wider internet
2. A Public relay node - sits between the private node and the Tezos network

Each of the above will have only the minimal software installed to run a Tezos node.  We will also configure a software firewall on each to ensure only the correct network connectivity is allowed.  We will also ensure the nodes are installed in appropriate security groups on our cloud provider to ensure another layer of security.