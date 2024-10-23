---
layout: post
title: "Signing HTTP Messages Burp Suite Extension"
---

I wrote the first PortSwigger [Burp Suite extension](https://github.com/nccgroup/HTTPSignatures) (see [blog post](https://www.nccgroup.com/us/research-blog/tool-release-httpsignatures-a-burp-suite-extension-implementing-http-signatures/)) that implements the [Signing HTTP Messages draft-ietf-httpbis-message-signatures-01](https://datatracker.ietf.org/doc/rfc9421/) specification draft document, for Burp users to seamlessly test applications that require HTTP Signatures. 

I disclosed and helped remediate HTTP Signatures-related vulnerabilities in Oracle Cloud, Peer-Tube, Nextcloud, Pleroma, Plume, Friendica, and Hubzilla. 
Two of these vulnerabilities in Nextcloud Social resulted in [CVE-2020-8278](https://hackerone.com/reports/921717) and [CVE-2020-8279](https://hackerone.com/reports/915585). 

Timeline:

- 2020-12-08 GitHub release: <https://github.com/nccgroup/HTTPSignatures>
- 2020-12-08 Blog post: <https://www.nccgroup.com/us/research-blog/tool-release-httpsignatures-a-burp-suite-extension-implementing-http-signatures/>
- 2020-07-12 Discovered an access control issue in the Nextcloud Social App: [CVE-2020-8278](https://hackerone.com/reports/921717)
- 2020-07-04 Discovered a lack of server certificate validation issue in the Nextcloud Social App: [CVE-2020-8279](https://hackerone.com/reports/915585)

