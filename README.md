# burry

## Use

The general usage is:

```bash
$ burry --help
Usage: burry [args]

Arguments:
  -b, --burryfest
        Create a burry manifest file .burryfest in the current directory.
        The manifest file captures the current command line parameters for re-use in subsequent operations.
  -c, --credentials string
        The credentials to use in format STORAGE_TARGET_ENDPOINT,KEY1=VAL1,...KEYn=VALn.
        Example: s3.amazonaws.com,ACCESS_KEY_ID=...,SECRET_ACCESS_KEY=...,BUCKET=...,PREFIX=...,SSL=...
  -e, --endpoint string
        The infra service HTTP API endpoint to use.
        Example: localhost:8181 for Exhibitor
  -f, --forget boolean 
        Forget existing data
  -i, --isvc string
        The type of infra service to back up or restore.
        Supported values are [etcd zk consul] (default "zk")
  -o, --operation string
        The operation to carry out.
        Supported values are [backup restore] (default "backup")
  -s, --snapshot string
        The ID of the snapshot.
        Example: 1483193387
  -t, --target string
        The storage target to use.
        Supported values are [local minio s3 tty local] (default "tty")
      --timeout=1: The infra service timeout, by default 1 second
  -v, --version
        Display version information and exit.
```

Note: If you want to re-use your command line parameters, use the `--burryfest` or `-b` argument: this creates a manifest file `.burryfest` in the current directory, capturing all your settings. If a manifest `.burryfest` exists in the current directory subsequent invocations use this and hence you can simply execute `burry`, without any parameters. Remove the `.burryfest` file in the current directory to reset stored command line parameters.

An example of a burry manifest file looks like:

```json
{
    "svc": "etcd",
    "svc-endpoint": "etcd.mesos:1026",
    "target": "local",
    "credentials": {
        "target-endpoint": "",
        "params": []
    }
}
```

Note that for every storage target other than `tty` a metadata file `.burrymeta` in the (timestamped) archive file will be created, something like:

```json
{
  "snapshot-date": "2016-12-31T14:52:42Z",
  "svc": "zk",
  "svc-endpoint": "leader.mesos:2181",
  "target": "s3",
  "target-endpoint": "s3.amazonaws.com"
}
```

### Backups

In general, since `--operation backup` is the default, the only required parameter for a backup operation is the `--endpoint`. That is, you'll have to provide the HTTP API of the ZooKeeper or etcd you want to back up:

```bash
$ burry --endpoint IP:PORT (--isvc etcd|zk) (--target tty|local|s3) (--credentials STORAGE_TARGET_ENDPOINT,KEY1=VAL1,...,KEYn=VALn)
```

Some concrete examples follow now.

#### Screen dump of local ZooKeeper content

To dump the content of a locally running ZK onto the screen, do the following:

```bash
# launching ZK:
$ docker ps
CONTAINER ID        IMAGE                                  COMMAND                  CREATED             STATUS              PORTS                                                                                            NAMES
9ae41a9a02f8        mbabineau/zookeeper-exhibitor:latest   "bash -ex /opt/exhibi"   2 days ago          Up 2 days           0.0.0.0:2181->2181/tcp, 0.0.0.0:2888->2888/tcp, 0.0.0.0:3888->3888/tcp, 0.0.0.0:8181->8181/tcp   amazing_kilby

# dump to screen:
$ DEBUG=true ./burry --endpoint localhost:2181
INFO[0000] Using burryfest /home/core/.burryfest  func=init
INFO[0000] My config: {InfraService:zk Endpoint:localhost:2181 StorageTarget:tty Creds:{StorageTargetEndpoint: Params:[]}}  func=init
INFO[0000] /zookeeper/quota:                             func=reapsimple
INFO[0000] Operation successfully completed.             func=main
```

See the [development and testing](dev.md#zookeeper) notes for the test setup.

#### Back up DC/OS system ZooKeeper to Amazon S3

To back up the content of the DC/OS system ZooKeeper (supervised by Exhibitor), do the following:

```bash
# let's first do a dry run:
$ ./burry --endpoint leader.mesos:2181
INFO[0000] My config: {InfraService:zk Endpoint:leader.mesos:2181 StorageTarget:tty Creds:{StorageTargetEndpoint: Params:[]}}  func=init
INFO[0006] Operation successfully completed.             func=main

# back up into Amazon S3:
$ ./burry --endpoint leader.mesos:2181 --target s3 --credentials s3.amazonaws.com,ACCESS_KEY_ID=***,SECRET_ACCESS_KEY=***
INFO[0000] My config: {InfraService:zk Endpoint:leader.mesos:2181 StorageTarget:s3 Creds:{InfraServiceEndpoint:s3.amazonaws.com Params:[{Key:ACCESS_KEY_ID Value:***} {Key:SECRET_ACCESS_KEY Value:***}]}}}  func=init
INFO[0008] Successfully stored zk-backup-1483166506/latest.zip (45464 Bytes) in S3 compatible remote storage s3.amazonaws.com  func=remoteS3
INFO[0008] Operation successfully completed. The snapshot ID is: 1483166506  func=main
```

#### Back up DC/OS system ZooKeeper to Local
```bash
# backup to local
$ ./burry --endpoint=zk-cluster.demo.svc:2181 --target=local
INFO[0000] Selected operation: BACKUP                    func=main
2024/03/27 10:26:39 Connected to 10.96.150.24:2181
2024/03/27 10:26:39 authenticated: id=144115192260002187, timeout=4000
2024/03/27 10:26:39 re-submitting `0` credentials after reconnect
INFO[0000] Rewriting root                                func=store
INFO[0000] Operation successfully completed. The snapshot ID is: 1711535199  func=main
```

See the [development and testing](dev.md#zookeeper) notes for the test setup. Note: in order to back up to Google Storage rather than to Amazon S3, use `--credentials storage.googleapis.com,ACCESS_KEY_ID=***,SECRET_ACCESS_KEY=***` in above command. Make sure that you have Google Storage as your default project and Interoperability enabled; see also settings in [console.cloud.google.com/storage/settings](https://console.cloud.google.com/storage/settings).

### Restores

For restores you MUST set `--operation restore` (or: `-o restore`) as well as provide a snapshot ID with `--snapshot/-s`. Note also that you CAN NOT restore from screen, that is, `--target/-t tty` is an invalid choice:

```bash
$ burry --operation restore --target local|s3 --snapshot sID (--isvc etcd|zk) (--credentials STORAGE_TARGET_ENDPOINT,KEY1=VAL1,...,KEYn=VALn)
```

Restoring ZooKeeper from local storage snashop
```bash
$ ./burry --endpoint=zk-cluster.demo.svc:2181 --target=local -i zk --operation=restore --snapshot=1711535199
INFO[0000] Selected operation: RESTORE                   func=main
INFO[0000] Connected to 10.96.150.24:2181               
INFO[0000] authenticated: id=144115192260002188, timeout=4000 
INFO[0000] re-submitting `0` credentials after reconnect 
INFO[0000] Operation successfully completed. Restored 0 items from snapshot 1711535199  func=main
```