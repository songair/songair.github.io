---
layout:              post
title:               Internal Structure Of Snapshot Repository
subtitle:            >
    Do you know these files?

lang:                en
date:                2021-09-04 10:11:10 +0200
categories:          [elasticsearch]
tags:                [elasticsearch, java]
comments:            true
excerpt:             >
    Do you know these files?

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

## Data Preparation

Next, let's take a look at some of the more important files.

But before getting started, we need to prepare the data locally: by starting Elasticsearch, and creating some data and snapshots.

```sh
es_repo="${HOME}/es-backup/blog-snapshot-repo-structure/"

# Start Elasticsearch
docker run \
  --rm \
  -p 9200:9200 \
  -p 9300:9300 \
  -e "discovery.type=single-node" \
  -e "cluster.name=es-docker-cluster" \
  -e "path.repo=/opt/elasticsearch/backup" \
  -v "${es_repo}:/opt/elasticsearch/backup" \
  docker.elastic.co/elasticsearch/elasticsearch:7.12.0

# Create document
curl -X PUT localhost:9200/my_index/_doc/1 \
  -H 'Content-Type: application/json' \
  -d '{"msg": "Hello Elasticsearch"}'

# Create snapshot repository
curl -X PUT localhost:9200/_snapshot/my_repo \
  -H 'Content-Type: application/json' \
  -d '{
  "type": "fs",
  "settings": {
    "location": "my_repo"
  }
}'

# Create snapshot
curl -X PUT localhost:9200/_snapshot/my_repo/my_snapshot_1 \
  -H 'Content-Type: application/json' \
  -d '{
  "indices": "my_index",
  "include_global_state": false,
  "metadata": {
    "taken_by": "Mincong",
    "taken_because": "https://mincong.io is the best blog for learning Elasticsearch"
  }
}'
```

Then enter the local folder `blog-snapshot-repo-structure`, make sure the repository `my_repo` and the files in it are ready: 

```
➜  blog-snapshot-repo-structure find my_repo
my_repo
my_repo/index-0
my_repo/indices
my_repo/indices/Uxom82JcSfORXgbtZ4jLSg
my_repo/indices/Uxom82JcSfORXgbtZ4jLSg/0
my_repo/indices/Uxom82JcSfORXgbtZ4jLSg/0/index-MwjmFzyOT_2NI6DdXLcsNw
my_repo/indices/Uxom82JcSfORXgbtZ4jLSg/0/__IlzZcMdOSkC-j6xx0Qj04A
my_repo/indices/Uxom82JcSfORXgbtZ4jLSg/0/snap-2hiUzvH3RPCp9iOeiTa6TQ.dat
my_repo/indices/Uxom82JcSfORXgbtZ4jLSg/0/__jQbQk7YYTwW8G5gL_RtR8w
my_repo/indices/Uxom82JcSfORXgbtZ4jLSg/meta-N1BHtXsBYxjWXi8lXhTR.dat
my_repo/index.latest
my_repo/meta-2hiUzvH3RPCp9iOeiTa6TQ.dat
my_repo/snap-2hiUzvH3RPCp9iOeiTa6TQ.dat
```

Now, we can further explore the contents of the repository! 

## File index-N


```sh
cat index-0 | jq
```

```js
{
  "min_version": "7.12.0",
  "uuid": "C87ijmZURAK3ij8MsAaDAw",
  "cluster_id": "aSNpPgDAShyYAiKhsun6IA",
  "snapshots": [
    {
      "name": "my_snapshot_1",
      "uuid": "2hiUzvH3RPCp9iOeiTa6TQ",
      "state": 1,
      "index_metadata_lookup": {
        "Uxom82JcSfORXgbtZ4jLSg": "Uz7B9HV2SJ6peiLiUMJhyg-_na_-1-2-1"
      },
      "version": "7.12.0"
    }
  ],
  "indices": {
    "my_index": {
      "id": "Uxom82JcSfORXgbtZ4jLSg",
      "snapshots": [
        "2hiUzvH3RPCp9iOeiTa6TQ"
      ],
      "shard_generations": [
        "MwjmFzyOT_2NI6DdXLcsNw"
      ]
    }
  },
  "index_metadata_identifiers": {
    "Uz7B9HV2SJ6peiLiUMJhyg-_na_-1-2-1": "N1BHtXsBYxjWXi8lXhTR"
  }
}
```

```
$ curl "localhost:9200/_cat/indices/my_index?v"
health status index    uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   my_index Uz7B9HV2SJ6peiLiUMJhyg   1   1          1            0      3.9kb          3.9kb
```

The above file `index-0` represents generation 0 of the snapshot repository. It is a `RepositoryData` serialized in JSON format, which contains all the snapshot IDs and their corresponding indexes. The UUID of the index `my_index` in the repository is `Uxom82JcSfORXgbtZ4jLSg`, and its corresponding UUID in the cluster is `Uz7B9HV2SJ6peiLiUMJhyg`. This index is referenced by a snapshot, which is the snapshot `my_snapshot_1` (`2hiUzvH3RPCp9iOeiTa6TQ`). 

## Conclusion

You can subscribe to the [feed of my blog](/feed.xml), follow me on [Twitter](https://twitter.com/mincong_h) or [GitHub](https://github.com/mincong-h/). Hope you enjoy this article, see you the next time!
