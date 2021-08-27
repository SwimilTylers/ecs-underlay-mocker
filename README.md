# ecs-underlay-mocker

Mocking a L2-underlay network over ECS L3-VPC.

```
underlayctl tool [-h|--help] [options] [other arguments]
 
tools:
ssh-no-password       allow master connecting to slave without password
master                master-client of underlay mocker, for slave deploying and config distributing
slave                 slave-client of underlay mocker, for concrete setting
```

The mocking process requires `ssh` and `scp` operations. `ssh-no-password` relates to 
ssh authentication, relieving you from tedious typing of passwords.

The `master` loads config and deploys `slave` across nodes automatically. 
In normal case, you ONLY have to care about `master` tool and underlay network config. 
Direct use of `slave` tool is NOT preferable, unless some error occurs.

## How to write an underlay config

We offer a sample yaml `config/net.yaml`.
1. declare the vxlan config used for underlay mocking, consisting of
    - `dev` device of underlay network
    - `parent` device of vpc network
    - `vnid` vnid of vxlan tunnel
    - `port` udp port of vxlan frame
    - `cidr` cidr of the underlay network
2. declare the nodes upon which we mock the underlay network, starting with `nodes`
    - `node` vpc ip of the node
    - `ip` underlay ip of the node, omit if empty
    - `gateway` whether the node is an underlay gateway, `no` if empty. 
      Once you set this field as `yes`, the `ip` must NOT be empty. 
      Otherwise, it will abort.

## How to apply an underlay config

1. copy `underlayctl` and your underlay network config to one of your VPC nodes
2. use tool `ssh-no-password` to generate ssh authentication.
3. use tool `master` to specify underlay network config. It will finish rest jobs.
   If you want to use the sample config, try this: `./underlayctl master --check-config config/net.yaml`.


## How to update an underlay config
**WARNING**: we do not support _delete-node-operation_.

You have to edit the config and resubmit it to `master`. To be mentioned, if you
update vxlan device config (e.g., vnid), you would better add `--force` option.

## Options for `underlayctl-master`
1. `--check-config` if set, it will display each config it loads and pause until
   you confirm the config. We recommend you to turn this option on because our yaml
   parser is not so robust, and manual checking will reduce the possibility of further
   misconfiguration.
2. `--force` if set, it will delete the underlay device before setting the device up.