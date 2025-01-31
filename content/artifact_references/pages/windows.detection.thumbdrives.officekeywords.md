---
title: Windows.Detection.Thumbdrives.OfficeKeywords
hidden: true
tags: [Client Event Artifact]
---

Users inserting Thumb drives or other Removable drive pose a
constant security risk. The external drive may contain malware or
other undesirable content. Additionally thumb drives are an easy way
for users to exfiltrate documents.

This artifact automatically scans any office files copied to a
removable drive for keywords. This could be useful to detect
exfiltration attempts of restricted documents.

We exclude very large removable drives since they might have too
many files.


```yaml
name: Windows.Detection.Thumbdrives.OfficeKeywords
description: |
  Users inserting Thumb drives or other Removable drive pose a
  constant security risk. The external drive may contain malware or
  other undesirable content. Additionally thumb drives are an easy way
  for users to exfiltrate documents.

  This artifact automatically scans any office files copied to a
  removable drive for keywords. This could be useful to detect
  exfiltration attempts of restricted documents.

  We exclude very large removable drives since they might have too
  many files.

type: CLIENT_EVENT

parameters:
  - name: officeExtensions
    default: "\\.(xls|xlsm|doc|docx|ppt|pptm)$"
    type: regex

  - name: yaraRule
    description: This yara rule will be run on document contents.
    type: yara
    default: |
      rule Hit {
        strings:
          $a = "this is my secret" wide nocase
          $b = "this is my secret" nocase

        condition:
          any of them
      }

sources:
  - query: |
        SELECT * FROM foreach(
          row = {
            SELECT * FROM Artifact.Windows.Detection.Thumbdrives.List()
            WHERE FullPath =~ officeExtensions
          },
          query = {
            SELECT * FROM Artifact.Generic.Applications.Office.Keywords(
              yaraRule=yaraRule, searchGlob=FullPath, documentGlobs="")
          })

```
