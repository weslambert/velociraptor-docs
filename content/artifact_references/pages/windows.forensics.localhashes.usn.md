---
title: Windows.Forensics.LocalHashes.Usn
hidden: true
tags: [Client Event Artifact]
---

This artifact maintains a local (client side) database of file
hashes. It is then possible to query this database using the
Generic.Forensic.LocalHashes.Query artifact


```yaml
name: Windows.Forensics.LocalHashes.Usn
description: |
  This artifact maintains a local (client side) database of file
  hashes. It is then possible to query this database using the
  Generic.Forensic.LocalHashes.Query artifact

type: CLIENT_EVENT

parameters:
  - name: PathRegex
    description: A regex to match the entire path (you can watch a directory or a file type).
    default: .
    type: regex

  - name: Device
    description: The NTFS drive to watch
    default: C:\\

  - name: HashDb
    description: Name of the local hash database
    default: hashdb.sqlite

  - name: SuppressOutput
    description: If this is set, the artifact does not return any rows to the server but will still update the local database.
    type: bool

  - name: UsnCheckPeriod
    type: int
    description: Dedup all file operations that occur within this period
    default: "10"

precondition: SELECT OS from info() where OS = "windows"

sources:
  - query: |
      -- Dont be too aggressive on the USN watching to conserve CPU usage
      LET NTFS_CACHE_TIME = 30
      LET USN_FREQUENCY = 60

      LET hash_db <= SELECT FullPath
      FROM Artifact.Generic.Forensic.LocalHashes.Init(HashDb=HashDb)

      LET path <= hash_db[0].FullPath

      LET _ <= log(message="Will use local hash database " + path)

      LET file_modifications = SELECT Device + FullPath AS FullPath
      FROM watch_usn(device=Device, accessor="ntfs")
      WHERE FullPath =~ PathRegex

      -- The USN journal may contain multiple entries for the same
      -- file modification (e.g. TRUNCATE followed by APPEND and
      -- CLOSE). We therefore dedup all entries that happen within the
      -- period as a single modification.
      LET deduped = SELECT * FROM foreach(row={
         SELECT * FROM clock(period=UsnCheckPeriod, start=0)
      },
      query={
         -- Each time the fifo is accessed we pull all the rows and
         -- dedup the path, then clear the cache.
         SELECT * FROM fifo(
             query=file_modifications,
             max_rows=5000,
             max_age=6000, flush=TRUE)
         GROUP BY FullPath
      })

      -- Stat each file that was changed to get its size and hash
      LET files = SELECT * FROM foreach(row=deduped,
      query={
         SELECT FullPath, Size, hash(path=FullPath).MD5 AS Hash, now() AS Time
         FROM stat(filename=FullPath)
         WHERE Mode.IsRegular
      })

      -- For each file hashed, insert to the local database
      LET insertion = SELECT FullPath, Hash, Size, Time, {
         SELECT * FROM sqlite(file=path,
            query="INSERT into hashes (path, md5, timestamp, size) values (?,?,?,?)",
            args=[FullPath, Hash, Time, Size])
      } AS Insert
      FROM files
      WHERE Insert OR TRUE

      // If output is suppressed do not emit a row, but still update the local database.
      SELECT FullPath, Hash, Size, Time
      FROM insertion
      WHERE NOT SuppressOutput

column_types:
  - name: Time
    type: timestamp

  - name: Hash
    type: hash

  - name: ClientId
    type: client_id

```
