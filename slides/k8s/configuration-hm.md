# Managing Configuration -hm

- Some applications need to be configured.

- Common ways for code to pick up configuration:
  - Command-line arguments
  - Environment variables
  - Configuration files

- How can we manage and inject configuration efficiently in Kubernetes?

---

## Passing Configuration to Containers

- Kubernetes provides multiple native mechanisms to pass configuration:
  - Command-line arguments (`command` and `args`)
  - Static environment variables (`env`)
  - Dynamic pod metadata via the **Downward API**
  - External configuration files and key-value pairs via **ConfigMaps**

---

## Command-line Arguments

- Pass options directly in the Pod specification template using `command` or `args`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: command-demo
spec:
  containers:
  - name: demo
    image: busybox
    command: ["ping"]
    args: ["-c", "5", "8.8.8.8"]