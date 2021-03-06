# Frequently Asked Questions (FAQ)

FAQ contains the frequently asked questions about two aspects:

- First, user-facing functionalities.
- Second, underlying concept and thoery.

Techinical question will not be included in FAQ.

## What is Dragonfly

**Dragonfly is an intelligent P2P based image and file distribution system.**

It aims to resolve issues related to low-efficiency, low-success rate and waste of network bandwidth in file transferring process. Especially in large-scale file distribution scenarios such as application distribution, cache distribution, log distribution, image distribution, etc.

In Alibaba, Dragonfly is invoked 2 Billion times and the data distributed is 3.4PB every month. Dragonfly has become one of the most important pieces of infrastructure at Alibaba.

While container technologies makes devops life easier most of the time, it sure brings a some challenges: the efficiency of image distribution, especially when you have to replicate image distribution on several hosts. Dragonfly works extremely well with both Docker and [PouchContainer](https://github.com/alibaba/pouch) for this scenario. It also is compatible with any other container formats.

It delivers up to 57 times the throughput of native docker and saves up to 99.5% the out bandwidth of registry(*2).

Dragonfly makes it simple and cost-effective to set up, operate, and scale any kind of files/images/data distribution.

## Is Dragonfly only designed for distribute container images

No, Dragonfly can be used to distribute all kinds of files, not only container images. For downloading files by Dragonfly, please refer to [Download a File](https://github.com/dragonflyoss/Dragonfly/blob/master/docs/quick_start/README.md#downloading-a-file-with-dragonfly). For pulling images by Dragonfly, please refer to [Pull an Image](https://github.com/dragonflyoss/Dragonfly/blob/master/docs/quick_start/README.md#pulling-an-image-with-dragonfly).

## What is the sequence of P2P distribution

Supernode will maintain a bitmap which records the correspondence between peers and pieces. When dfget starts to download, supernode will return several pieces info (4 in default）according to the scheduler.

**NOTE**: The scheduler will decide whether to download from the supernode or other peers. As for the details of the scheduler, please refer to [scheduler algorithm](#what-is-the-peer-scheduling-algorithm-in-default)

## How do supernode and peers manage file cache which is ready for other peer's pulling

Supernode will download files and cache them via CDN. For more information, please refer to [The sequence of supernode's CDN functionality](#what-is-the-sequence-of-supernode's-cdn-functionality).

After finishing distributing file from other peers, `dfget` should do two kinds of thing:

- first, construct all the pieces into unioned file(s);
- second, backup the unioned file(s) in configured directory, by default "$HOME/.small-dragonfly/data" with a suffix of ".server";
- third, move the original unioned file(s) to the destination path.

**NOTE**: The supernode cannot manage the cached file which is already distributed to peer node at all. For that what will happen if the cached file on a peer is deleted, please refer to [What if you kill the dfget server process or delete the source files](#what-if-you-kill-the-dfget-server-process-or-delete-the-source-files)

## What is the sequence of supernode's CDN functionality

When dfget register a task to supernode, supernode will check whether the file to be downloaded has a cache locally.

If the file to be downloaded which has not been cached in supernode yet, supernode will do as the following sequence:

- First Step: supernode triggers the downloading task asynchronously:
  - fetch the file/image length;
  - divide the length into pieces;
  - start to download the file piece by piece and store it locally;
- Second Step:
  - supernode finishes to download one piece
  - supernode starts the P2P distribution work for the downloaded one piece among peers;

If the requested file which has already been cached in supernode, supernode will send an HTTP GET request, which contains both HTTP headers `If-None-Match:<eTag>` and `If-Modified-Since:<lastModified>`, to source file network address to determine whether the remote file has been updated.

In addition, supernode does not have to wait for all the piece downloadings finished, so it can concurrently start piece downloading once one piece downloaded.

## What if you kill the dfget server process or delete the source files

If a file on the peer is deleted manually or by GC, the supernode won't know that. And in the subsequent scheduling, if multiple download tasks fail from this peer, the scheduler will add it to blacklist. So do with that if the server process be killed or other abnormal conditions.

## What is the peer scheduling algorithm in default

- Distribute the number of pieces evenly. Select the piece with the smallest number in the entire P2P network so that the distribution of each piece in the P2P network is balanced to avoid "Nervous resources".

- Nearest distance priority. For a peer, the piece closest to the piece currently being downloaded is preferentially selected, so that the peer can achieve the effect of sequential read and write approximately which will improve the I/O efficiency of file.

- Local blacklist and global blacklist. An example is easier to understand: When peer A fails to download from peer B, B will become the local blacklist of A, and then the download tasks of A will filter B out; When the number of failed downloads from B reaches a certain threshold, B will become the global blacklist, and all the download tasks will filter B out.

- Self-isolation. PeerA will only download files from the supernode after it fails to download multiple times from other peers, and will also be added to the global blacklist. So the peer A will no longer interact with peers other than the supernode.

- Peer load balancing. This mechanism will control the number of upload pieces and download pieces that each peer can provide simultaneously and the priority as the target peer.

## What is the size of block(piece) when distribution

Dragonfly tries to make block(piece) size dynamically to ensure efficiency.

The size of pieces which is calculated as per the following strategy:

- If file's total size is less than 200MB, then the piece size is `4MB` by default.
- Otherwise, it equals to `min{ totalSize/100MB + 2 MB, 15MB }`.

## What is the difference between Dragonfly's P2P algorithm and bit-torrent(BT)

Dragonfly's P2P algorithm and bit-torrent(BT) are both the implementation of peer-to-peer protocol. For the difference between them, we describe them in the following table

|Aspect|Dragonfly|Bit-Torrent(BT)|
|:-:|:-:|:-:|
|seed's dynamically compress| support| no support|
|resume from break-point|support|no support|
|dynamically block size setting|support, see [question](#what-is-the-size-of-blockpiece-when-distribution)| no support(block size is fixed)|
|transparent of seed to client|support(user only needs to provide URL of file)|no support(BT needs to generate seed in tracker first, and then provide it to client) |

## What is the policy of bandwidth limit in peer network

We can enforce network bandwidth limit on both peer nodes and supernode of Dragonfly cluster.

For peer node itself, Dragonfly can set network bandwidth limit for two parts:

- one single downloading task; by default, limit is 10MB/s per downloading task. User can set task network bandwidth limit by parameter `--locallimit` within `dfget`.
- the whole network bandwidth consumed by P2P distribution on the peer node. This can be configured via parameter `totallimit` within `dfget`. If user has set `--totallimit=40M`, then both TX and RX limit are 40 MB/s.

For supernode, Dragonfly allows user to limit both the input and output network bandwidth.

You can also config it with config files, refer to [link](./docs/cli_reference/dfget.md#etcdragonflyconf)

## Why does dfget still keep running after file/image distribution finished

In a P2P network, a peer not only plays a role of downloader, but also plays a role of uploader.

- For downloader, peer needs to download block/piece of the file/image from other peers (supernode is one special peer as well);
- For uploader, peer needs to provide block/piece downloading service for other peers. At this time, other peers downloads block/piece from this peer, and this peer can be treated as an uploader.

Back to question, when a peer finishes to download file/image, while there may be other peers which is still downloading block/piece from this peer. Then this dfget process still exist for the potential uploading service.

Only when there is no new task to download file/images or no other peers coming to download block/piece from this peer, the dfget process will terminate.

## Do I have to change my container engine configuration to support Dragonfly

Currently Dragonfly supports almost all kinds of container engines, such as Docker, [PouchContainer](https://github.com/alibaba/pouch). When using Dragonfly, only one part of container engine's configuration needs update. It is the `registry-mirrors` configuration. This configuration update aims at making image pulling request from container engine will all be sent to `dfdaemon` process locally. And dfget does the request translation thing and proxies it to `dfget` locally.

Configure container engine's `registry-mirrors` is quite easy. We take docker as an example. Administrator should modify configuration file of docker `/etc/docker/daemon.json` to add the following item:

```yml
"registry-mirrors": ["http://127.0.0.1:65001"]
```

> Note: please remember restarting container engine after updating configuration.

## Do we support HA of supernode in Dragonfly

Currently no. Supernode in Dragonfly suffers the single node of failure. In the later release, we will try to realise the HA of Dragonfly.

## How to use Dragonfly in Kubernetes

It is very easy to deloy Dragonfly in Kubernetes with [Helm](https://github.com/helm/helm). For more information of Dragonfly's Helm Chart, please refer to project [dragonflyoss/helm-chart](https://github.com/dragonflyoss/helm-chart).

## Can an image from a third-party registry be pulled via Dragonfly

We **CANNOT** pull an image from a third-party registry via Dragonfly. Images from third-party registry means that the name of image has no registry address, for example, image `a.b.com/admin/mysql:5.6` is a third-party image, and it is from registry `a.b.com`. To the opposite, `admin/mysql:5.6` is not from third-party registry, because name of the image does not contain a registry address.

Because administrator needs to configure `--registry-mirrors=["http://127.0.0.1:65001"]` for container engine to proxy part of image pulling requests to `dfdaemon` which listens on 65001 locally, images not from a third-part registry will use this registry mirror. However, when user pulls image `a.b.com/admin/mysql:5.6` which contains a third-party registry address, the container engine will directly access the third-party registry to pull the image, ignoring the configured registry mirror. Then It has nothing to do with the `dfdaemon` within Dragonfly.

## Where is the log directory of Dragonfly client dfget

Log is very important for debugging or tracing. `dfget`'s log directory is `$HOME/.small-dragonfly/logs/dfclient.log`.

## Where is the data directory of Dragonfly client dfget

The meta data directory of `dfget` is `$HOME/.small-dragonfly/meta`.

This directory caches the address list of the local node (IP, hostname, IDC, security domain, and so on), managing nodes, and supernodes.

When started for the first time, the Dragonfly client will access the managing node. The managing node will retrieve the security domains, IDC, geo-location, and more information of this node by querying armory, and assign the most suitable supernode. The managing node also distributes the address lists of all other supernodes. dfget stores the above information in the meta directory on the local drive. When the next job is started, dfget reads the meta information from this directory, avoiding accessing the managing node repeatedly.

dfget always finds the most suitable assigned node to register. If fails, it requests other supernodes in order. If all fail, then it requests the managing node once again, updates the meta information, and repeats the above steps. If still fails, then the job fails.

> Note: If the dfclient.log suggests that the supernode registration fails all the time, try delete all files from the meta directory and try again.

## What is temp directory of dfget and dfdaemon

The default temp data directory of `dfget` is `$HOME/.small-dragonfly/data` which cannot be configured at present.

The default temp data directory of `dfdaemon` is `$HOME/.small-dragonfly/dfdaemon/data` which can be configured by flag `--localrepo`.

When `dfdaemon` downloads images by `dfget`, it will set the target file path of `dfget` which  is `the temp data directory of dfdaemon`. So `dfget` will download the file to `$HOME/.small-dragonfly/data` and move to `$HOME/.small-dragonfly/dfdaemon/data` after downloaded totally.

## How to clean up the data directory of Dragonfly

Normally, Dragonfly automatically cleans up files which are not accessed in three minutes from the uploading file list, and the residual files that live longer than one hour from the data directory.

Follow these steps to clean up the data directory manually:

- Identify the account to which the residual files belong.
- Check if there is any running dfget process started by other accounts.
- If there is such a process, then terminate the process, and start a dfget process with the account identified at Step 1. This process will automatically clean up the residual files from the data directory.

## Does Dragonfly support pulling images from an HTTPS enabled registry

Currently, Dragonfly **DOES NOT** support pulling images from an HTTPS enabled registry.

`dfdaemon` is a proxy between container engine and registry. It captures every request sent from container engine to registry, filters out all the image layer downloading requests and uses `dfget` to download them.

We should investigate HTTPS proxy implementation and make it possible for dfdaemon to get HTTP data from tcp data, analysis them and download image layers correctly.

We are planning to support this feature in Dragonfly 0.5.0.

## Does Dragonfly support pulling an private image which needs username/password authentication

We are planning to support this feature in Dragonfly 0.5.0.