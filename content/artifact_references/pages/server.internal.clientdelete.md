---
title: Server.Internal.ClientDelete
hidden: true
tags: [Server Event Artifact]
---

An internal queue that receives events when a client is deleted.


```yaml
name: Server.Internal.ClientDelete
description: |
  An internal queue that receives events when a client is deleted.

type: SERVER_EVENT

column_types:
  - name: ClientId
    description: The client that was deleted.
  - name: Principal
    description: The principal who initiated the deletion.

```
