---
title: Linux.Sys.CPUTime
hidden: true
tags: [Client Artifact]
---

Displays information from /proc/stat file about the time the cpu
cores spent in different parts of the system.


```yaml
name: Linux.Sys.CPUTime
description: |
  Displays information from /proc/stat file about the time the cpu
  cores spent in different parts of the system.
parameters:
  - name: procStat
    default: /proc/stat
sources:
  - precondition: |
      SELECT OS From info() where OS = 'linux'
    query: |
        LET raw = SELECT * FROM split_records(
           filenames=procStat,
           regex=' +',
           columns=['core', 'user', 'nice', 'system',
                    'idle', 'iowait', 'irq', 'softirq',
                    'steal', 'guest', 'guest_nice'])
        WHERE core =~ 'cpu.+'

        SELECT core AS Core,
               atoi(string=user) as User,
               atoi(string=nice) as Nice,
               atoi(string=system) as System,
               atoi(string=idle) as Idle,
               atoi(string=iowait) as IOWait,
               atoi(string=irq) as IRQ,
               atoi(string=softirq) as SoftIRQ,
               atoi(string=steal) as Steal,
               atoi(string=guest) as Guest,
               atoi(string=guest_nice) as GuestNice FROM raw

```
