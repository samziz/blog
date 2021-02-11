---
layout: post
title: "You've got mail"
date: 2020-10-11
categories: Foundations
---

The SMTP protocol, used to send email, is a phenomenally successful protocol; it's one of the earliest parts of the Internet to have survived virtually unaltered since the Soviet Union still existed. Email has undergone a few changes - IMAP was introduced to allow us to have email clients on several devices, DKIM/SPF was introduced to guarantee provenance and [accidentally non-repudiation](https://blog.cryptographyengineering.com/2020/11/16/ok-google-please-publish-your-dkim-secret-keys/) - but SMTP by virtue of its simplicity has remained the sole way that email mailbox servers communicate with each other.

## The SMTP handshake

The process of sending mail begins with a protocol negotiation which is surprisingly sophisticated, for an ancient and resilient protocol, and closer to the TLS protocol negotiation than it is to simpler protocols like TCP or BGP.
