# K3S in Docker on Mac

A quick start running K3S cluster in docker container on a Mac machine.


Pre-request:
- [Docker (Docker destop)](https://www.docker.com/)
- [Kubectl](https://kubernetes.io/docs/reference/kubectl/)

## Compose file for K3S

Reference to the offical repo for the [docker-compolse](https://github.com/k3s-io/k3s/blob/master/docker-compose.yml) file.

Spin up K3S server and agent:

```sh
$ docker-compose up -d
```

A `kubeconfig.yaml` file should be generated automatically.

Check whether the cluster has been started correctly:

```sh
$ kubectl --kubeconfig=./kubeconfig.yaml get pods -A
```

## Compose file for local Docker Registry - Optional

Play with local docker registry. This is an optional part.

- Local docker registry

Spin up the local `docker registry` by:

```sh
$ docker-compose -f ./docker-compose-registry.yaml -d
```

A `docker registry v2` instance should be running at `5000` port.

If you want to push/pull image from the local registry, you have to add a `insecure-registries` in Docker Desktop due to HTTP secure issue.

Docker -> Preferences -> Docker Engine

Add `insecure-registries` config then apply and restart docker.

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "features": {
    "buildkit": true
  },
  "insecure-registries": [
    "register.local:5000"
  ]
}
```

- Setup Local Domain For The Registry

Typically, the local registry service works well with localhost. Sometimes, it would be better to have a domain that is convenient for the K3s cluster pulling image from the registry service. 

```
sudo vim /etc/hosts
```

Then add host resolving the name

```sh
127.0.0.1    registry.local
```

- Push docker image to local Registry

Re-tag an image and verify the local registry works fine for us.

Pull `busybox` image as it's really small

```sh
$ docker pull busybox
```

Re-tag it to `register.local:5000/my-busybox`:

```sh
$ docker image tag busybox registry.local:5000/my-busybox
```

Push it to local registry

```
$ docker push register.local:5000/my-busybox
```

You should see pushed message in the console. 

Another check is to run a CRUL command:

```sh
$ curl -X GET http://registry.local:5000/v2/_catalog

{"repositories":["my-busybox"]}
```

- Integrate with K3S

Mount registry config file `registries.yaml` on each K3S node (every node you want to pull image from local registry).

Append the volume mount snippet in the K3S `docker-compose` file.

```yaml
volumes:
    - ./registries.yaml:/etc/rancher/k3s/registries.yaml # Need to be added for each nodes
```

- Deploy A Pod in the K3s Cluster with Local Registry

```sh
$ docker-compose -f ./docker-compose.yaml -f ./docker-compose-registry.yaml up

docker push registry.local:5000/my-busybox

kubectl --kubeconfig=./kubeconfig.yaml run my-busybox -it --rm  --image=registry.local:5000/my-busybox
```

After the `kubectl run` command run, everything works well if you see a command prompt.
