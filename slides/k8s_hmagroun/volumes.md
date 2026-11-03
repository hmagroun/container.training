# Volumes

- Volumes are special directories that are mounted in containers

- Volumes can have many different purposes:

  - share files and directories between containers running on the same machine

  - share files and directories between containers and their host

  - centralize configuration information in Kubernetes and expose it to containers

  - manage credentials and secrets and expose them securely to containers

  - store persistent data for stateful services

  - access storage systems (like Ceph, EBS, NFS, Portworx, and many others)

---

## Volumes ≠ Persistent Volumes

- Volumes and Persistent Volumes are related, but very different!

- *Volumes*:

  - appear in Pod specifications (we'll see that in a few slides)

  - do not exist as API resources (**cannot** do `kubectl get volumes`)

- *Persistent Volumes*:

  - are API resources (**can** do `kubectl get persistentvolumes`)

  - correspond to concrete volumes (e.g. on a SAN, EBS, etc.)

  - cannot be associated with a Pod directly; but through a Persistent Volume Claim

  - won't be discussed further in this section

---

## Adding a volume to a Pod

- We will start with the simplest Pod manifest we can find

- We will add a volume to that Pod manifest

- We will mount that volume in a container in the Pod

- By default, this volume will be an `emptyDir`

  (an empty directory)

- It will "shadow" the directory where it's mounted

---

## Our basic Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-without-volume
spec:
  containers:
  - name: nginx
    image: nginx
```

This is a MVP! (Minimum Viable Pod😉)

It runs a single NGINX container.

---

## Adding a volume

- We need to add the volume in two places:

  - at the Pod level (to declare the volume)

  - at the container level (to mount the volume)

- We will declare a volume named `www`

- No type is specified, so it will default to `emptyDir`

  (as the name implies, it will be initialized as an empty directory at pod creation)

- In that pod, there is also a container named `nginx`

- That container mounts the volume `www` to path `/usr/share/nginx/html/`

---

## The Pod with a volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-volume
spec:
  volumes:
  - name: www
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: www
      mountPath: /usr/share/nginx/html/
```

---

## Volume lifecycle

- The lifecycle of a volume is linked to the pod's lifecycle

- This means that a volume is created when the pod is created

- This is mostly relevant for `emptyDir` volumes

  (other volumes, like remote storage, are not "created" but rather "attached" )

- A volume survives across container restarts

- A volume is destroyed (or, for remote storage, detached) when the pod is destroyed

