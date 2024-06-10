# 2024-06-11 msg-auth-status

[![Discord chat][discord-badge]][discord-url]

![meme improving email security should not be a hard choice to make]((https://cdn.jsdelivr.net/gh/pinkforest/pinkforest@main/assets/meme_email_security_8tfuz2.jpg)

I've just published crate [msg-auth-status] that I'm using in [arewe.at] for e-mail pipelines using my [imap connector].

This crate provides parsing for e-mail headers related to Domainkeys Identified Mail (DKIM) described in [RFC 6376] as well as Message Authentication Status as described in [RFC 8601].

## Why ?

E-mail ecosystem is in shambles and needs a lot of love both from standardisation as well as tools implementation sides.

See the recording of my talk [Your e-mail is now your database?](https://www.youtube.com/watch?v=rDl6JzeaZXQ) a few months back about e-mail I put together hurrily in 15 minutes about the pipeline I was building.

Pretty much [not a single provider](https://www.uriports.com/blog/dkim-ed25519-adoption/) either provides or even verifies ed25519-sha256 DKIM Signatures.

Comparing to HTTP stack e-mail has not had as much push to for example getting mandatory required TLS use and has not modernised itself.

Quite a lot of the HTTP standardisation improvements since 1.1 was driven by the browser ecosystem as the applications were moving more into using web environment despite it's own challenges.

This is one of my efforts to provide implementation targeted into some of the difficult parts of the standard that can be done wrong.

In the long run if we provide better implementations that are easy to use and plug into we might be able to improve the current landscape.

## Intended Use ?

This enables downstream e.g. e-mail clients, processors and what not else to hold the mail server accountable that it sets these headers given it is now easier to use these headers.

## Difficulties

These headers are really difficult given implementations differ widely and the intepretation of the Request For Comments (RFC) also vary widely.

Some of the RFCs allow for example comments anywhere in the headerfield making parsing the field more difficult in static compiled languages.

With Perl this could have been couple of regular expressions if you take the tradeoff of being potentially getting your regular expressions blown out by malicious actor.

Parsing these headers correctly in Rust is no easy task and I still have quite few bugs -

This is why I use data driven testing samples from real use of variety of providers, property testing (thanks @ThomasEizinger for your upcoming PR you have not sent me yet), fuzzing and benchmarking.

## Future work

I still need to write higher level API for a verifier that provides mis-use resistant API to do this reasonably correct as much as both the standards and the current server implementations allow.

And I need to get my e-mail client [funnelweb] to use it outside the pipeline for [arewe.at]

If this work helps you, please consider helping out including providing funding to improve the current e-mail landscape.

You can send me an e-mail to: pinkforest_edu .. the AT thing in between ....  pm.me - or use the e-mail in my GitHub bio and ROT13 decode it.

[msg-auth-status]: https://github.com/areweat/msg-auth-status
[arewe.at]: https://github.com/areweat/
[imap connector]: https://github.com/fluvio-connectors/imap-connector
[RFC 6376]: https://www.rfc-editor.org/rfc/rfc6376.html
[RFC 8601]: https://datatracker.ietf.org/doc/html/rfc8601
[funnelweb]: https://github.com/pinkforest/funnelweb

[discord-badge]: https://img.shields.io/discord/934761553952141402.svg?logo=discord
[discord-url]: https://discord.gg/rXVsmzhaZa
