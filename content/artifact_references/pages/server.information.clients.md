---
title: Server.Information.Clients
hidden: true
tags: [Server Artifact]
---

This artifact returns the total list of clients, their hostnames and
the last times they were seen.

We also include a list of usernames on this machine, as gathered by
the last Windows.Sys.Users artifact that was collected. Note that
the list of usernames may be outdated if that artifact was not
collected recently.


```yaml
name: Server.Information.Clients
description: |
  This artifact returns the total list of clients, their hostnames and
  the last times they were seen.

  We also include a list of usernames on this machine, as gathered by
  the last Windows.Sys.Users artifact that was collected. Note that
  the list of usernames may be outdated if that artifact was not
  collected recently.

type: SERVER

sources:
  - query: |
        SELECT client_id,
               os_info.fqdn as HostName,
               os_info.system as OS,
               os_info.release as Release,
               timestamp(epoch=last_seen_at/ 1000000).String as LastSeenAt,
               last_ip AS LastIP,
               last_seen_at AS _LastSeenAt
        FROM clients(count=100000)
        ORDER BY _LastSeenAt DESC

```
