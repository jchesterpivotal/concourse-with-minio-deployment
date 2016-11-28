# concourse-minio

Quick n dirty examples of how to add minio as an S3 alternative in your Concourse BOSH deployment.

- `concourse-minio-deployment`: A deployment which sets up both Concourse and Distributed Minio.
- `minio-pipeline.yml`: A demo pipeline showing how to use minio as if were an S3 resource.
- `cloud-config.yml`: A pretty standard cloud config, you probably have one already. This one speaks the BOSH-lite dialect.

## Setup

I assume you have installed and configured a BOSH director (I used BOSH-lite).

### Upload releases

First, follow the instructions in Concourse docs to upload Concourse releases

Next, do this to upload Minio:

    git clone git@github.com:shinji62/minio-dist-boshrelease.git
    cd minio-dist-boshrelease
    bosh upload release releases/minio-dist/minio-dist-1.yml

Your `bosh releases` should look vaguely like this:

    +-------------+----------+-------------+
    | Name        | Versions | Commit Hash |
    +-------------+----------+-------------+
    | concourse   | 2.5.0*   | 5186fc5     |
    | garden-runc | 1.0.3*   | c6c4c73c    |
    | minio-dist  | 1*       | 78b9331f+   |
    +-------------+----------+-------------+

### Deploy

First, edit `(((REPLACE ME)))` placeholders in `bosh/concourse-minio-deployment.yml`.

You might also need to edit for vm types, network name etc to match your cloud config.

For key ID and secret key, the actual value seems largely unimportant, so long as it has the dimensions of an AWS key. I used a random string generator for mine.

Then:

    bosh deployment bosh/concourse-minio-deployment.yml
    bosh deploy

This starts up all the bits. There are four minio servers, because that's the minimum permissible in distributed mode.

### Setting up a bucket

You can't do this from inside Concourse, so you need to make your own. [Download the mc CLI](https://www.minio.io/downloads/#minio-client).

Find the external IP of one of the BOSH VMs running minio:

    bosh vms

    ...
    minio-server/0 (3c0e1ea5-c548-4fc9-84bf-1420df0bf17c) | running | z1 | default-vm | 10.244.0.6
    ...

It doesn't matter which one, since they already find each other and any of them will respond to S3 requests.

Then use it to set up `mc` to talk to that server:

    ./mc config host add minio-server-0 http://10.244.0.6:9000 [key ID] [secret key]

Note that the IP is the same. The `:9000` is necessary or it won't work.

`mc` is annoying, it won't complain about *anything* and will silently try to work. This means you can easily be talking to a localhost copy of minio without realising it.

It doesn't work like BOSH. I am not talking to a single minio set of servers, I need to specify it when I'm working with `mc`. Otherwise everything is routed to the localhost server and the next few steps don't work.

OK, let's set up a bucket:

    ./mc mb minio-server-0/minio-example-bucket
    ./mc policy both minio-server-0/minio-example-bucket

Note the `minio-server-0/` prefix. Without this, you create a local bucket, which is not what you want (it's not what anyone wants, IMO).

### Using the bucket

Refer to `pipelines/minio-pipeline.yml`, again replace the placeholders.

In particular, the `access_key_id` and `secret_access_key` need to match the ones in your deployment.

The magic points are:
1. `endpoint` is pointed to a worker with an IP. This is currently hardcoded into the pipeline. Dynamic magic TBD. You might do a DNS server somewhere.
2. `bucket` will do the right thing here without a servername prefix, because `s3-resource` is smarter than `mc`.
3. `disable_ssl` is true because for now, I can't be bothered working out the key shuttling.

### And that should be it.

I hope.