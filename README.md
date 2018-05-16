# State Repository for Eirini-Director-Dev

## Working with the director locally

### Prereqs

- Install [Bosh-CLI](https://bosh.io/docs/cli-v2/)
- Get Softlayer VPN Access

### Auth to director

1. Connect with VPN!

1. Update you /etc/hosts file with the following entry:

  ```
  <eirini-dev-ip> eirini-dev.softlayer.com
  ```

1. Export auth variables


   ```
   $ export BOSH_CLIENT=admin && export BOSH_CLIENT_SECRET=`bosh int ./environments/softlayer/director/eirini-dev/vars.yml --path /admin_password`
   ```

1. Alias you environment (if not happend already)

   ```
   bosh -e eirini-dev.softlayer.com --ca-cert <(bosh int ./environments/softlayer/director/eirini-dev/vars.yml --path /director_ssl/ca) alias-env eirini-dev)
   ```

1. Do a quick test with `bosh -e eirini-dev vms`


### SSH to some cf component

1. Setup ssh tunnel

  ```
  $ ssh -4 -D 5001 -fNC jumpbox@eirini-dev.softlayer.com -i ./environments/softlayer/director/eirini-dev/jumpbox.key
  ```

1. Use BOSH_ALL_PROXY env variable to ssh 

  ```
  $ BOSH_ALL_PROXY=socks5://localhost:5001 bosh -e eirini-dev -d cf ssh cube
  ```

## Initial Director Setup

When a director was setup, you will need to add two `iptable` rules to the NAT table, which redirect requests to the eirni `registry` and `sync` process:

1. SSH to the director 

  ```
  $ ssh root@eirini-dev.softlayer.com
  ```

  INFO: Password can be looked up in SL
  INFO: In case you get a "too many authentication failures for user root", you will need clear ssh hosts: `$ ssh-add -D`

1. Update iptables

```
# SYNC
$ iptables -t nat -I PREROUTING 2 -p tcp --dport 8090 -j DNAT --to 10.244.0.142:8085

# REGISTRY
iptables -t nat -I PREROUTING 2 -p tcp --dport 8089 -j DNAT --to 10.244.0.142:8080
```

## Initial Kubernetes Setup

As long as we use our own eirni registry (which is not secure), we will need to add insecure-registry to the worker nodes. This can be done by running a container with privileged rights and the `/etc` direcotry of the host bin mounted. You can do that by running the following pod:

```
apiVersion: v1
kind: Pod
metadata:
  name: privileged
spec:
  securityContext:
    runAsUser: 0
    fsGroup: 0
  volumes:
  - name: etc
    hostPath:
      path: /etc
      type: Directory
  containers:
  - name: etc-try
    command:
    - "/bin/bash"
    - "-c"
    - "--"
    args:
    - "while true; do sleep 30; done;"
    image: ubuntu
    volumeMounts:
    - name: etc
      mountPath: /workspace/etc
    securityContext:
      allowPrivilegeEscalation: true
      privileged: true
      runAsUser: 0
```

Login to the kubernetes cluster and run:

```
$ kubectl apply -f <theYamlFile>
```

Attach to the running container:

```
$ kubectl exec -it privileged /bin/bash
```

Go to `/workspace/etc/docker/` and add the registry to the daemon.json (if it not exists create it):

```
{
  "insecure_registries":["registry-endpoint"]
}
```

## Login to CF 

### Prereqs

- Install CF CLI 

### Login

1. Auth as admin using the admin password

```
cf auth admin $(bosh int ./cf-deployment/deployment-vars.yml --path /cf_admin_password)
```

1. If not happend, create a org and a space for eirini

  ```
  $ cf create-org eirini 
  $ cf target -o eirini 
  $ cf create-space dev
  $ cf target -s dev 
  ```
  otherwise you can simply target org and dev by executing:

  ```
  $ cf target -o eirini -s dev 
  ```
