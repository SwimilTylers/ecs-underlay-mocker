# ecs-underlay-mocker

Mocking an L2-underlay network over ECS L3-VPC.

```
underlayctl tool [-h|--help] [options] [other arguments]
 
tools:
ssh-auth     authenticate ssh connection towards those nodes
master       master-client of underlay mocker, for slave deploying and config distributing
slave        slave-client of underlay mocker, for concrete setting
lite         lightweight client of underlay mocker without slave deploying
```

The mocking process requires `ssh` and `scp` operations. `ssh-auth` relates to 
ssh authentication, relieving you from tedious typing of passwords.

The `master` loads config and deploys `slave` across nodes automatically. 
In normal case, you ONLY have to care about `master` tool and underlay network config. 
Direct use of `slave` tool is NOT preferable, unless some error occurs.

The `lite` is a lightweight client for underlay mocker and all configuration relies on
`ssh` remote calls without `scp` delivery of slave clients. You can use it as a quick start.

## Configure a simple underlay network

1. copy `underlayctl` to one of your VPC nodes
2. (if necessary) use tool `ssh-auth` to grant ssh authentication.
3. use tool `lite` and specify VPC nodes. It will generate underlay network config 
   automatically and finish the rest jobs.

For other parameters, seek `./underlayctl lite -h` for details.

### IP Allocation

If you want to set up an underlay network among these vpc nodes
`172.19.18.228/30, 172.19.18.229/30, 172.19.18.230/30`,  try this cmd
```shell
./underlayctl lite 172.19.18.228 172.19.18.229 172.19.18.230
```
If you want to assign underlay ip to those nodes automatically, please specify 
CIDR of the underlay network.
```shell
./underlayctl lite --cidr=192.168.56.0/24 172.19.18.228 172.19.18.229 172.19.18.230
```
Therefore, we get config on each node (vpc iface default as `eth0`, underlay iface default as `eth1`):
- `eth0=172.19.18.228/30, eth1=192.168.56.1/24`
- `eth0=172.19.18.229/30, eth1=192.168.56.2/24`
- `eth0=172.19.18.230/30, eth1=192.168.56.3/24`

Tool `lite` allocates IP according to the order of your NODEs.

### Update Configuration

**WARNING**: we do not support _delete-node-operation_.

Tool `lite` will brutally delete specified underlay network device (default as `eth1`) on each node if exists before 
any configuration. If you want to **add nodes**, or **assign/delete/reallocate underlay ip**, redo the command
straightforward.

Set up an underlay network through:

```shell
./underlayctl lite --cidr=192.168.56.0/24 172.19.18.228 172.19.18.229
```
Thus, we get
- `eth0=172.19.18.228/30, eth1=192.168.56.1/24`
- `eth0=172.19.18.229/30, eth1=192.168.56.2/24`

To add a node `172.19.18.230` to the network, run:

```shell
./underlayctl lite --cidr=192.168.56.0/24 172.19.18.228 172.19.18.229 172.19.18.230
```
Thus, we get:
- `eth0=172.19.18.228/30, eth1=192.168.56.1/24`
- `eth0=172.19.18.229/30, eth1=192.168.56.2/24`
- `eth0=172.19.18.230/30, eth1=192.168.56.3/24`

### Multiple Networks

If you want to install multiple underlay networks, you have to specify the underlay network device through
`--underlay-dev` parameter explicitly. The VNID of VxLAN is automatically allocated (201-209) unless you specify it explicitly 
through `--net-id` parameter. 

```shell
# install an underlay network on eth1, vnid is auto-configured to 201
./underlayctl lite --underlay-dev=eth1 --cidr=192.168.56.0/24 172.19.18.230 172.19.18.228 172.19.18.229
# install another underlay network on eth2, vnid is auto-configured to 202
./underlayctl lite --underlay-dev=eth2 --cidr=192.168.56.0/24 172.19.18.230 172.19.18.231 172.19.18.232
```

## Configure a more complicated network

### How to write an underlay config

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

### How to apply an underlay config

1. copy `underlayctl` and your underlay network config to one of your VPC nodes
2. (if necessary) use tool `ssh-auth` to generate ssh authentication.
3. use tool `master` to specify underlay network config. It will finish rest jobs.
   
If you want to use the sample config, try this: `./underlayctl master --check-config using config/net.yaml`.


### How to update an underlay config
**WARNING**: we do not support _delete-node-operation_.

You have to edit the config and resubmit it to `master`. To be mentioned, if you
update vxlan device config (e.g., vnid), you would better add `--force` option.

```shell
./underlayctl master --check-config --force using config/net.yaml
```

## Man Page for `underlayctl-master`

There are two ways of usage:
```
underlayctl master [options] using [files]
underlayctl master [options] auto [parameters] [nodes]
```
### Options
1. `--check-config` if set, it will display each config it loads and pause until
   you confirm the config. We recommend you to turn this option on because our yaml
   parser is not so robust, and manual checking will reduce the possibility of further
   misconfiguration.
2. `--force` if set, it will delete the underlay device before setting the device up.

### Using config from files

If you want to configure several underlay networks at one time, choose this way.

### Using auto-generated config

We can automatically generate underlay network config for given nodes, allowing integration
into other testing tools. You can specify some parameters, or using default ones. Client `master` will pass
these parameters to `slave` clients accordingly.

To be mentioned, if you specify CIDR of the underlay network, we will automatically
assign underlay ip to each node. If you want more specified config, please choose aforementioned loaded config.

```
underlayctl master auto [parameters] [nodes]
 
parameters:
-h, --help                   show brief help
--cidr=CIDR                  specify the cidr of the underlay network, omit if empty
--underlay-dev=DEV_NAME      designate a name for the device on underlay network, default as eth1
--parent-dev=DEV_NAME        specify the parent device from which underlay device is derived, default as eth0
--net-id=ID                  specify the vnid for vxlan tunnel for underlay traffic, default as 100
--udp-port=PORT              specify the udp port of vxlan tunnel for underlay traffic, default as 4096
```