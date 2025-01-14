---
title: Linux.Remediation.Quarantine
hidden: true
tags: [Client Artifact]
---

This artifact applies quarantine to Linux systems via nftables.
It expects the target system to have nftables installed, and
hence the availability of nft CLI.

This artifact will create a table, with the default name
*vrr_quarantine_table*, which contains three chains. One
for inbound traffic, one for outbound traffic, and the other
for forwarding traffic. The chains will cut off all traffics
except those for DNS lookup and velociraptor itself.

To unquarantine the system, set the *RemovePolicy* parameter to *True*.


```yaml
name: Linux.Remediation.Quarantine
description: |
  This artifact applies quarantine to Linux systems via nftables.
  It expects the target system to have nftables installed, and
  hence the availability of nft CLI.

  This artifact will create a table, with the default name
  *vrr_quarantine_table*, which contains three chains. One
  for inbound traffic, one for outbound traffic, and the other
  for forwarding traffic. The chains will cut off all traffics
  except those for DNS lookup and velociraptor itself.

  To unquarantine the system, set the *RemovePolicy* parameter to *True*.

precondition: SELECT OS From info() where OS = 'linux'

type: CLIENT

required_permissions:
  - EXECVE

parameters:
  - name: pathToNFT
    default: /usr/sbin/nft
    description: We depend on nft to manage the tables, chains, and rules.

  - name: TableName
    default: vrr_quarantine_table
    description: Name of the quarantine table

  - name: MessageBox
    description: |
        Optional message box notification to send to logged in users. 256
        character limit.

  - name: RemovePolicy
    type: bool
    description: Tickbox to remove policy.

sources:
  - query: |
     // If a MessageBox configured truncate to 256 character limit
     LET MessageBox <= parse_string_with_regex(
               regex='^(?P<Message>.{0,255}).*',
               string=MessageBox).Message

     // Parse a URL to get domain name.
     LET get_domain(URL) = parse_string_with_regex(
          string=URL, regex='^https?://(?P<Domain>[^:/]+)').Domain

     // Parse a URL to get the port
     LET get_port(URL) = if(condition= URL=~"https://[^:]+/", then="443",
           else=if(condition= URL=~"http://[^:]+/", then="80",
           else=parse_string_with_regex(string=URL,
               regex='^https?://[^:/]+(:(?P<Port>[0-9]*))?/').Port))

     // extract Velociraptor config for policy
     LET extracted_config <= SELECT * FROM foreach(
               row=config.server_urls,
               query={
                   SELECT
                       get_domain(URL=_value) AS DstAddr,
                       get_port(URL=_value) AS DstPort,
                       'VelociraptorFrontEnd' AS Description,
                       _value AS URL
                   FROM scope()
               })

     // delete table
     LET delete_table_cmd = (pathToNFT, 'delete', 'table', 'inet', TableName)

     // add table
     LET add_table_cmd = (pathToNFT, 'add', 'table', 'inet', TableName)

     // add inbound chain
     LET add_inbound_chain_cmd = (
          pathToNFT, 'add', 'chain', 'inet', TableName, 'inbound_chain',
          '{', 'type', 'filter', 'hook', 'input', 'priority', '0\;', 'policy', 'drop\;', '}')

     // add udp rule inbound chain to allow DNS lookups
     LET add_udp_rule_to_inbound_chain_cmd = (
          pathToNFT,'add','rule','inet', TableName, 'inbound_chain',
          'udp', 'sport', 'domain',
          'ct', 'state', 'established', 'accept')

     // add outbound chain
     LET add_outbound_chain_cmd = (
          pathToNFT, 'add', 'chain', 'inet', TableName, 'outbound_chain',
          '{', 'type', 'filter', 'hook', 'output', 'priority', '0\;', 'policy', 'drop\;', '}')

     // add tcp rule outbound chain to allow DNS traffics
     LET add_tcp_rule_to_outbound_chain_cmd = (
          pathToNFT, 'add', 'rule', 'inet', TableName, 'outbound_chain',
          'tcp', 'dport', '{', '53', '}',
          'ct', 'state', 'new,established', 'accept')

     // add udp rule outbound chain to allow DNS and DHCP traffics
     LET add_udp_rule_to_outbound_chain_cmd = (
          pathToNFT,'add','rule','inet', TableName, 'outbound_chain',
          'udp', 'dport', '{', '53,67,68', '}',
          'ct', 'state', 'new,established', 'accept')

     // add forward chain
     LET add_forward_chain_cmd = (
          pathToNFT, 'add', 'chain', 'inet', TableName, 'forward_chain',
          '{', 'type', 'filter', 'hook', 'forward', 'priority', '0\;', 'policy', 'drop\;', '}')


     // delete quarantine table
     LET delete_quarantine_table = SELECT
          timestamp(epoch=now()) as Time,
          TableName + ' table removed.' AS Result
       FROM execve(argv=delete_table_cmd, length=10000)

     // add quarantine table
     LET add_quarantine_table = SELECT
          timestamp(epoch=now()) as Time,
          combine_results(Stdout=Stdout, Stderr=Stderr,
          ReturnCode=ReturnCode,
          Message=TableName + ' added.') AS Result
       FROM execve(argv=add_table_cmd, length=10000)

     // add inbound chain
     LET add_inbound_chain = SELECT
          timestamp(epoch=now()) as Time,
          combine_results(Stdout=Stdout, Stderr=Stderr,
          ReturnCode=ReturnCode,
          Message='Added inbound_chain to ' + TableName + ' table.') AS Result
       FROM execve(argv=add_inbound_chain_cmd, length=10000)

     // add tcp rule to inbound_chain to allow connections from Velociraptor
     // FIXME(gye): may need to add IPv6 rules if DstAddr is an IPv6 address
     LET add_velociraptor_rule_to_inbound_chain = SELECT * FROM foreach(
          row={
              SELECT DstAddr, DstPort from extracted_config
          },
          query={
              SELECT
                  timestamp(epoch=now()) as Time,
                  combine_results(Stdout=Stdout, Stderr=Stderr,
                  ReturnCode=ReturnCode,
                  Message='Added tcp rule to inbound_chain in ' + TableName + ' table.') AS Result
              FROM execve(argv=(
                  pathToNFT, 'add', 'rule', 'inet', TableName, 'inbound_chain',
                  'ip', 'saddr', DstAddr, 'tcp', 'sport', '{', DstPort, '}',
                  'ct', 'state', 'established', 'accept'), length=10000)
          })

     // add udp rule to inbound_chain to allow DNS lookups
     LET add_udp_rule_to_inbound_chain = SELECT
          timestamp(epoch=now()) as Time,
          combine_results(Stdout=Stdout, Stderr=Stderr,
          ReturnCode=ReturnCode,
          Message='Added udp rule to inbound_chain in ' + TableName + ' table.') AS Result
       FROM execve(argv=add_udp_rule_to_inbound_chain_cmd, length=10000)

     // add inbound chain
     LET add_outbound_chain = SELECT
          timestamp(epoch=now()) as Time,
          combine_results(Stdout=Stdout, Stderr=Stderr,
          ReturnCode=ReturnCode,
          Message='Added outbound_chain to ' + TableName + ' table.') AS Result
       FROM execve(argv=add_outbound_chain_cmd, length=10000)

     // add tcp rule to inbound_chain to allow connections from Velociraptor
     // FIXME(gye): may need to add IPv6 rules if DstAddr is an IPv6 address
     LET add_velociraptor_rule_to_outbound_chain = SELECT * FROM foreach(
          row={
              SELECT DstAddr, DstPort from extracted_config
          },
          query={
              SELECT
                  timestamp(epoch=now()) as Time,
                  combine_results(Stdout=Stdout, Stderr=Stderr,
                  ReturnCode=ReturnCode,
                  Message='Added tcp rule to inbound_chain in ' + TableName + ' table.') AS Result
              FROM execve(argv=(
                  pathToNFT, 'add', 'rule', 'inet', TableName, 'outbound_chain',
                  'ip', 'daddr', DstAddr, 'tcp', 'dport', '{', DstPort, '}',
                  'ct', 'state', 'established,new', 'accept'), length=10000)
          })

     // add tcp rule to outbound_chain to allow DNS traffics
     LET add_tcp_rule_to_outbound_chain = SELECT
          timestamp(epoch=now()) as Time,
          combine_results(Stdout=Stdout, Stderr=Stderr,
          ReturnCode=ReturnCode,
          Message='Added tcp rule to outbound_chain in ' + TableName + ' table.') AS Result
       FROM execve(argv=add_tcp_rule_to_outbound_chain_cmd, length=10000)

     // add udp rule to outbound_chain to allow DNS and DHCP traffics
     LET add_udp_rule_to_outbound_chain = SELECT
          timestamp(epoch=now()) as Time,
          combine_results(Stdout=Stdout, Stderr=Stderr,
          ReturnCode=ReturnCode,
          Message='Added udp rule to outbound_chain in ' + TableName + ' table.') AS Result
       FROM execve(argv=add_udp_rule_to_outbound_chain_cmd, length=10000)

     // add forward chain
     LET add_forward_chain = SELECT
          timestamp(epoch=now()) as Time,
          combine_results(Stdout=Stdout, Stderr=Stderr,
          ReturnCode=ReturnCode,
          Message='Added forward_chain to ' + TableName + ' table.') AS Result
       FROM execve(argv=add_forward_chain_cmd, length=10000)

     // test connection to a frontend server
     LET test_connection = SELECT * FROM foreach(
         row={
             SELECT DstAddr, DstPort from extracted_config
         },
         query={
             SELECT *
                 Url,
                 response
             FROM
                 http_client(url='https://' + DstAddr + ':' + DstPort + '/server.pem',
                     disable_ssl_security='TRUE')
             WHERE Response = 200
             LIMIT 1
         })

     // final check to keep or remove policy
     // TODO(gyee): for now we are using the wall commmand to send the message.
     // Will need to look into using libnotify instead.
     LET final_check = SELECT * FROM if(condition= test_connection,
               then={
                   SELECT
                       timestamp(epoch=now()) as Time,
                       if(condition=MessageBox,
                           then= TableName + ' connection test successful. MessageBox sent.',
                           else= TableName + ' connection test successful.'
                       ) AS Result
                    FROM if(condition=MessageBox,
                        then= {
                            SELECT * FROM execve(argv=['wall',MessageBox])
                        },
                        else={
                            SELECT * FROM scope()
                        })
               },
               else={
                   SELECT
                       timestamp(epoch=now()) as Time,
                       TableName + ' failed connection test. Removing quarantine table.' AS Result
                   FROM delete_quarantine_table
               })


     // Execute content
     SELECT * FROM if(condition=RemovePolicy,
               then=delete_quarantine_table,
               else={
                   SELECT * FROM chain(
                       a=delete_quarantine_table,
                       b=add_quarantine_table,
                       c=add_inbound_chain,
                       d=add_velociraptor_rule_to_inbound_chain,
                       e=add_udp_rule_to_inbound_chain,
                       f=add_outbound_chain,
                       g=add_velociraptor_rule_to_outbound_chain,
                       h=add_tcp_rule_to_outbound_chain,
                       i=add_udp_rule_to_outbound_chain,
                       j=add_forward_chain,
                       k=final_check)
               })


```
