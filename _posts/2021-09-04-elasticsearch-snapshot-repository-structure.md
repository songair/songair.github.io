---
layout:              post
title:               Internal Structure Of Snapshot Repository
subtitle:            >
    Given one sentence to expand the title or explain why this article may interest your readers.

lang:                en
date:                2021-09-04 10:11:10 +0200
categories:          [elasticsearch]
tags:                [elasticsearch, java]
comments:            true
excerpt:             >
    TODO
image:               /assets/bg-dmitrij-paskevic-YjVa-F9P9kk-unsplash.jpg
cover:               /assets/bg-dmitrij-paskevic-YjVa-F9P9kk-unsplash.jpg
article_header:
  type:              overlay
  theme:             dark
  background_color:  "#203028"
  background_image:
    gradient:        "linear-gradient(135deg, rgba(0, 0, 0, .6), rgba(0, 0, 0, .4))"
wechat:              false
ads:                 none
---

This article is translated with Google Translate and reviewed by Mincong.
{:.info}

## Introduction

If you use an Elasticsearch cluster in production, I believe you must have heard of Elasticsearch's [snapshot and restore feature](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/snapshot-restore-apis.html), because it is an important means to ensure that cluster data is not lost. There are a lot of materials on the Internet about how to use Elasticsearch snapshots, but there are very few articles about its implementation. Today, I want to discuss with you the internal structure of the Elasticsearch snapshot repository. Knowing it can give us a better understanding of how Elasticsearch's snapshots work, and can also provide more ideas for troubleshooting when there is a problem in production.

After reading this article, you will understand:

- What is a snapshot repository?
- Different files inside a snapshot repository
- Learning more about `index-N` files
- Learning more about the `index.latest` file
- Learning more about snapshot information files

Without further ado, let's get started right away!

## What is a snapshot repository?

A snapshot repository is a location where Elasticsearch stores snapshots. One snapshot repository can store multiple snapshots. Snapshots are how Elasticsearch backups your data. You can create a snapshot for one single index or multiple indices. Inside a snapshot repository, snapshots are incremental: new snapshots will only snapshot those parts that were not snapshotted in the previous snapshot to avoid wasting time and storage space. There are many types of snapshot repositories: they can be local repository or remote repositories for  cloud providers, such as AWS S3, Google Cloud Storage, Azure Blob Storage, Aliyun OSS, etc.

## Different files inside a snapshot repository

Below, let's take a look the different files stored inside a snapshot repository. Here is an excerpt from the [Javadoc of Elasticsearch 7.12 about Snapshot Repository](https://github.com/elastic/elasticsearch/blob/7.12/server/src/main/java/org/elasticsearch/repositories/blobstore/package-info.java):

```
 STORE_ROOT
 |- index-N           - JSON serialized {@link org.elasticsearch.repositories.RepositoryData} containing a list of all snapshot ids
 |                      and the indices belonging to each snapshot, N is the generation of the file
 |- index.latest      - contains the numeric value of the latest generation of the index file (i.e. N from above)
 |- incompatible-snapshots - list of all snapshot ids that are no longer compatible with the current version of the cluster
 |- snap-20131010.dat - SMILE serialized {@link org.elasticsearch.snapshots.SnapshotInfo} for snapshot "20131010"
 |- meta-20131010.dat - SMILE serialized {@link org.elasticsearch.cluster.metadata.Metadata } for snapshot "20131010"
 |                      (includes only global metadata)
 |- snap-20131011.dat - SMILE serialized {@link org.elasticsearch.snapshots.SnapshotInfo} for snapshot "20131011"
 |- meta-20131011.dat - SMILE serialized {@link org.elasticsearch.cluster.metadata.Metadata } for snapshot "20131011"
 .....
 |- indices/ - data for all indices
    |- Ac1342-B_x/ - data for index "foo" which was assigned the unique id Ac1342-B_x (not to be confused with the actual index uuid)
    |  |             in the repository
    |  |- meta-20131010.dat - JSON Serialized {@link org.elasticsearch.cluster.metadata.IndexMetadata} for index "foo"
    |  |- 0/ - data for shard "0" of index "foo"
    |  |  |- __1                      \  (files with numeric names were created by older ES versions)
    |  |  |- __2                      |
    |  |  |- __VPO5oDMVT5y4Akv8T_AO_A |- files from different segments see snap-* for their mappings to real segment files
    |  |  |- __1gbJy18wS_2kv1qI7FgKuQ |
    |  |  |- __R8JvZAHlSMyMXyZc2SS8Zg /
    |  |  .....
    |  |  |- snap-20131010.dat - SMILE serialized {@link org.elasticsearch.index.snapshots.blobstore.BlobStoreIndexShardSnapshot} for
    |  |  |                      snapshot "20131010"
    |  |  |- snap-20131011.dat - SMILE serialized {@link org.elasticsearch.index.snapshots.blobstore.BlobStoreIndexShardSnapshot} for
    |  |  |                      snapshot "20131011"
    |  |  |- index-123         - SMILE serialized {@link org.elasticsearch.index.snapshots.blobstore.BlobStoreIndexShardSnapshots} for
    |  |  |                      the shard (files with numeric suffixes were created by older versions, newer ES versions use a uuid
    |  |  |                      suffix instead)
    |  |
    |  |- 1/ - data for shard "1" of index "foo"
    |  |  |- __1
    |  |  |- index-Zc2SS8ZgR8JvZAHlSMyMXy - SMILE serialized {@code BlobStoreIndexShardSnapshots} for the shard
    |  |  .....
    |  |
    |  |-2/
    |  ......
    |
    |- 1xB0D8_B3y/ - data for index "bar" which was assigned the unique id of 1xB0D8_B3y in the repository
    ......
```

Organize them again in a table:

File path | explanation
:--- | :---
`index-N` | `RepositoryData` serialized in JSON format, including all snapshot IDs and their corresponding indexes. N represents the generation of this file.
`index.latest` | The file is a pointer that represents the last generation index file in digital form, which is the number N mentioned above. Here N is a hexadecimal number, for example, the index of the 100th generation (decimal), and finally represented by 64 in hexadecimal, because 64 = 16*6 + 4.
`incompatible-snapshots` | A list of all snapshot IDs that are no longer compatible with the current cluster version
`snap-20131010.dat` | `SnapshotInfo` serialized in SMILE format, used to represent the information corresponding to the snapshot 20131010
`meta-20131010.dat` | `Metadata` serialized in SMILE format, used to represent the metadata corresponding to the snapshot 20131010  (includes only global metadata)
`indices/` | Data for all indices
`indices/Ac1342-B_x/` | Index the data corresponding to foo. The UUID of the index in the repository is Ac1342-B_x. But do not confuse with the UUID of the index.
`indices/Ac1342-B_x/meta-20131010.dat` | Index foo `IndexMetadata` serialized in JSON format
`indices/Ac1342-B_x/0/` | Index foo data corresponding to shard 0
`indices/Ac1342-B_x/0/__1` | Files ending in numbers were created by an older version of Elasticsearch
`indices/Ac1342-B_x/0/__2` | Same as above
`Indices / Ac1342-B_x / 0 / __ VPO5oDMVT5y4Akv8T_AO_A` | segment files, with specific mappings real segment, see `snap-*` file
`indices/Ac1342-B_x/0/__1gbJy18wS_2kv1qI7FgKuQ` | Same as above
`indices/Ac1342-B_x/0/__R8JvZAHlSMyMXyZc2SS8Zg` | Same as above
`indices/Ac1342-B_x/0/snap-20131010.dat` | Snapshot 20131010 `BlobStoreIndexShardSnapshot` serialized in SMILE format
`indices/Ac1342-B_x/0/snap-20131011.dat` | Snapshot 20131011 `BlobStoreIndexShardSnapshot` serialized in SMILE format
`indices/Ac1342-B_x/0/index-123` | Shard 0 `BlobStoreIndexShardSnapshots` serialized in SMILE format. Files with numeric suffixes were created by older versions, newer ES versions use a uuid suffix instead

## Conclusion

You can subscribe to the [feed of my blog](/feed.xml), follow me on [Twitter](https://twitter.com/mincong_h) or [GitHub](https://github.com/mincong-h/). Hope you enjoy this article, see you the next time!
