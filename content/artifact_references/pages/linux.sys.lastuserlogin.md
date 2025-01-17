---
title: Linux.Sys.LastUserLogin
hidden: true
tags: [Client Artifact]
---

Find and parse system wtmp files. This indicate when the user last logged in.

```yaml
name: Linux.Sys.LastUserLogin
description: Find and parse system wtmp files. This indicate when the
             user last logged in.
parameters:
  - name: wtmpGlobs
    default: /var/log/wtmp*

    # This is automatically generated from dwarf symbols by Rekall:
    # gcc -c -g -o /tmp/test.o /tmp/1.c
    # rekall dwarfparser /tmp/test.o

    # And 1.c is:
    # #include "utmp.h"
    # struct utmp x;

  - name: MaxCount
    default: 10000
    type: int64

export: |
  LET wtmpProfile <= '''
       [
         ["Header", 0, [
           ["records", 0, "Array", {
               "type": "utmp",
               "count": "x=>MaxCount",
               "max_count": "x=>MaxCount"
           }]
         ]],

         ["timeval", 8, [
          ["tv_sec", 0, "int32"],
          ["tv_usec", 4, "int32"]
         ]],

         ["utmp", 384, [
          ["ut_host", 76, "String", {
           "length": 256
          }],

          ["ut_tv", 340, "timeval"],

          ["ut_id", 40, "String", {
           "length": 4
           }],

          ["ut_type", 0, "Enumeration", {
            "type": "short int",
            "choices": {
               "0": "EMPTY",
               "1": "RUN_LVL",
               "2": "BOOT_TIME",
               "5": "INIT_PROCESS",
               "6": "LOGIN_PROCESS",
               "7": "USER_PROCESS",
               "8": "DEAD_PROCESS"
             }
          }],

          ["ut_user", 44, "String", {
           "length": 32
          }]
        ]]
       ]
   '''

sources:
  - precondition: |
      SELECT OS From info() where OS = 'linux'
    query: |
      LET parsed = SELECT FullPath, parse_binary(
                   filename=FullPath,
                   profile=wtmpProfile,
                   struct="Header"
                 ) AS Parsed
      FROM glob(globs=split(string=wtmpGlobs, sep=","))

      SELECT * FROM foreach(row=parsed,
      query={
         SELECT * FROM foreach(row=Parsed.records,
         query={
           SELECT FullPath, ut_type AS Type,
              ut_id AS ID,
              ut_host as Host,
              ut_user as User,
              timestamp(epoch=ut_tv.tv_sec) as login_time
          FROM scope()
        })
      })

```
