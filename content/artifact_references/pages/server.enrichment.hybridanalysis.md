---
title: Server.Enrichment.HybridAnalysis
hidden: true
tags: [Server Artifact]
---

Submit a file hash to Hybrid Analysis for a verdict. Default free API restriction is 200 requests/min or 2000 requests/hour.

This artifact can be called from within another artifact (such as one looking for files) to enrich the data made available by that artifact.

Ex.

  `SELECT * from Artifact.Server.Enrichment.HybridAnalysis(Hash=$YOURHASH)`


```yaml
name: Server.Enrichment.HybridAnalysis
author: Wes Lambert -- @therealwlambert
description: |
  Submit a file hash to Hybrid Analysis for a verdict. Default free API restriction is 200 requests/min or 2000 requests/hour.

  This artifact can be called from within another artifact (such as one looking for files) to enrich the data made available by that artifact.

  Ex.

    `SELECT * from Artifact.Server.Enrichment.HybridAnalysis(Hash=$YOURHASH)`

type: SERVER

parameters:
    - name: Hash
      type: string
      description: The file hash to submit to Hybrid Analysis (MD5, SHA1, SHA256).
      default:

    - name: HybridAnalysisKey
      type: string
      description: API key for Hybrid Analysis. Leave blank here if using server metadata store.
      default:

    - name: UserAgent
      type: string
      description: Name of the user agent used for submitting hashes.
      default: Velociraptor

sources:
  - query: |
        LET Creds = if(
           condition=HybridAnalysisKey,
           then=HybridAnalysisKey,
           else=server_metadata().HybridAnalysisKey)

        LET URL <= 'https://hybrid-analysis.com/api/v2/search/hash'

        LET Data = SELECT parse_json_array(data=Content) as Content
        FROM http_client(
            url=URL,
            headers=dict(`api-key`=Creds,
                         `user-agent`=UserAgent,
                         `Content-Type`="application/x-www-form-urlencoded"),
            params=dict(hash=Hash),
            method='POST')

        SELECT * from foreach (
            row=Data,
            query={
                SELECT Content as _Content,
                       Content.verdict[0] as Verdict
                FROM scope()
            })

```
