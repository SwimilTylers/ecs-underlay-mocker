# ecs-underlay-mocker

Mocking an L2-underlay network over ECS L3-VPC.

```
underlayctl tool [-h|--help] [options] [other arguments]
 
tools:
ssh-auth              authenticate ssh connection towards those nodes
describe              describe config of the underlay network
install               mocking an underlay network using vxlan tunnel
```

The mocking process requires `ssh` operations. `ssh-auth` relates to 
ssh authentication, relieving you from tedious typing of passwords. The `descibe` 
tool displays the (expected) config of the underlay network, while `install` carries
out the config.
- If you are unfamiliar with the underlay network config, use `describe` to 
  check it out.
- If some failure occurs during the config, add `-v` when running `install`. It 
  prints out the actual commands.
  
When you use this tool, follow these steps:
1. copy `underlayctl` to somewhere connects your VPC nodes
2. (if necessary) use tool `ssh-auth` to grant ssh authentication.
3. run tool `install` and specify VPC nodes. It will generate underlay network config
   automatically and finish the rest jobs.
   
## Demos
1. If you want to set up an underlay network among these vpc nodes
   `172.19.18.228/30, 172.19.18.229/30, 172.19.18.230/30`, try this cmd:
   ```shell
   ./underlayctl install 172.19.18.228 172.19.18.229 172.19.18.230
   ```
2. If you want to allocate underlay network **IP** (`192.168.56.0/24`) among nodes, 
   add the flag `--cidr=192.168.56.0/24`:
   ```shell
   ./underlayctl install --cidr=192.168.56.0/24 172.19.18.228 172.19.18.229 172.19.18.230
   ```
3. If you want to specify the UNIQUE underlay network **gateway** (`172.19.18.228`), you have to add the prefix
   to `gw:` the node:
   ```shell
   ./underlayctl install --cidr=192.168.56.0/24 gw:172.19.18.228 172.19.18.229 172.19.18.230
   ```
4. If you want to **connect** the underlay network **to another network** (`10.96.0.0/12`) via the gateway (`172.19.18.228`),
   add the flag `--add-route-via-gw=10.96.0.0/12`:
   ```shell
   ./underlayctl install --cidr=192.168.56.0/24 --add-route-via-gw=10.96.0.0/12 gw:172.19.18.228 172.19.18.229 172.19.18.230
   ```
5. If you want to **add a node** (`172.19.18.231`) right now, append it to the of the last cmd directly:
   ```shell
   ./underlayctl install --cidr=192.168.56.0/24 --add-route-via-gw=10.96.0.0/12 gw:172.19.18.228 172.19.18.229 172.19.18.230 172.19.18.231
   ```

You can replace `install` with `describe` to check the config at any step:
```
# Print by the following cmd:
# ./underlayctl describe --cidr=192.168.56.0/24 --add-route-via-gw=10.96.0.0/12 gw:172.19.18.228 172.19.18.229 172.19.18.230 172.19.18.231

[NODES, cidr=192.168.56.0/24, vnid=201, port=8472]
[0] [eth0=172.19.18.228, eth1=192.168.56.1/24, gateway]
[1] [eth0=172.19.18.229, eth1=192.168.56.2/24]
[2] [eth0=172.19.18.230, eth1=192.168.56.3/24]
[3] [eth0=172.19.18.231, eth1=192.168.56.4/24]
[MORE ROUTES]
[0] [10.96.0.0/12 dev eth1 via 192.168.56.1 onlink]
```

## Aonther Demo: two connected network for k8s cluster

I have five vpc nodes: `172.19.18.232, 172.19.18.229, 172.19.18.230, 172.19.18.231, 172.19.18.228`

I want set up two underlay networks. Both connect to each other via `172.19.18.232`.
+ `192.168.56.0/24`: consisting of `172.19.18.232, 172.19.18.229, 172.19.18.230`
+ `192.168.57.0/24`: consisting of `172.19.18.232, 172.19.18.231, 172.19.18.228`

I want to install k8s clusters upon both the underlay networks with kubeadm. I use default service cidr (`10.96.0.0/12`).
On node `172.19.18.232`, run the following commands:

```shell
./underlayctl install --cidr=192.168.56.0/24 --add-route-to=10.96.0.0/12 --add-route-via-gw=192.168.57.0/24 gw:172.19.18.232 172.19.18.229 172.19.18.230
./underlayctl install --cidr=192.168.57.0/24 --add-route-to=10.96.0.0/12 --add-route-via-gw=192.168.56.0/24 gw:172.19.18.232 172.19.18.231 172.19.18.228
```

## More options
```
underlayctl-install - mocking an underlay network using vxlan tunnel

underlayctl install [-h|--help]
underlayctl install [-v|--verbose] [parameters] [nodes]
underlayctl install [-v|--verbose] [parameters] [gw:gateway_node] [other_nodes]
 
parameters:
-h, --help                   show brief help
-v, --verbose                show the content of remote calls
--cidr=CIDR                  specify the cidr of the underlay network, omit if empty
--gateway-ip=IP              specify the underlay ip of the gateway, auto-configured if cidr is assigned
--add-route-to=CIDR          add an additional route to underlay network device
--add-route-via-gw=CIDR      add an additional route via underlay network gateway
--underlay-dev=DEV_NAME      designate a name for the device on underlay network, default as eth1
--parent-dev=DEV_NAME        specify the parent device from which underlay device is derived, default as eth0
--net-id=ID                  specify the vnid for vxlan tunnel for underlay traffic, auto-configured if empty
--udp-port=PORT              specify the udp port of vxlan tunnel for underlay traffic, default as 8472
```
