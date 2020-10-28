## Setting up the Kubernetes Frontend Load Balancer

These steps are performed on the `loadbalancer` server

You will need to load balance usage of the Kubernetes API across multiple control nodes. In this section , we create a simple nginx server to perform this balancing. Once complete, we will be able to interact with both
control nodes of your kubernetes cluster using the nginx load balancer.

Install `HAProxy` on the server
```
sudo apt-get update && sudo apt-get install -y haproxy
```

loadbalancer# cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg 
frontend kubernetes
    bind 172.16.0.30:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server controller-0 172.16.0.20:6443 check fall 3 rise 2
    server controller-1 172.16.0.40:6443 check fall 3 rise 2
EOF
```
Now that we have the new configuration in place, reload nginx  with the new configuration
```
sudo service haproxy restart
```
##### Verification
```
curl -k https://localhost:6443/version
```
output
```
{
  "major": "1",
  "minor": "19",
  "gitVersion": "v1.19.2",
  "gitCommit": "f5743093fd1c663cb0cbc89748f730662345d44d",
  "gitTreeState": "clean",
  "buildDate": "2020-09-16T13:32:58Z",
  "goVersion": "go1.15",
  "compiler": "gc",
  "platform": "linux/arm64"
 )
```
