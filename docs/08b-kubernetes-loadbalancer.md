## Setting up the Kubernetes Frontend Load Balancer

These steps are performed on the `loadbalancer server`

You will need to load balance usage of the Kubernetes API across multiple control nodes. In this section , we create a simple nginx server to perform this balancing. Once complete, we will be able to interact with both
control nodes of your kubernetes cluster using the nginx load balancer.

Load nginx on the server
```
sudo apt-get install -y nginx
sudo systemctl enable nginx
sudo mkdir -p /etc/nginx/tcpconf.d
sudoedit /etc/nginx/nginx.conf
```
Add the following to the end of nginx.conf:
```
include /etc/nginx/tcpconf.d/*;
```
Set up some environment variables for the lead balancer config file:

```
CONTROLLER0_IP=<controller 0 private ip>
CONTROLLER1_IP=<controller 1 private ip>
```
**Example**
```
CONTROLLER0_IP=171.16.0.20
CONTROLLER1_IP=171.16.0.40
```

Create the load balancer nginx config file:
```
cat << EOF | sudo tee /etc/nginx/tcpconf.d/kubernetes.conf
stream {
    upstream kubernetes {
        server $CONTROLLER0_IP:6443;
        server $CONTROLLER1_IP:6443;
    }

    server {
        listen 6443;
        listen 443;
        proxy_pass kubernetes;
    }
}
EOF
```
Now that we have the new configuration in place, reload nginx  with the new configuration
```
sudo nginx -s reload
```
##### Verification
```
curl -k https://localhost:6443/version
```
output
```
{
  "major": "1",
  "minor": "18",
  "gitVersion": "v1.18.6",
  "gitCommit": "dff82dc0de47299ab66c83c626e08b245ab19037",
  "gitTreeState": "clean",
  "buildDate": "2020-07-15T16:51:04Z",
  "goVersion": "go1.13.9",
  "compiler": "gc",
  "platform": "linux/arm64"
}
```
