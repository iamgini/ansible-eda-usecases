# Event-Driven-Kubernetes-with-Ansible

Pre-req

```shell
$ pip install kubernetes
```

Install ansible-rulebook

```shell
dnf --assumeyes install java-17-openjdk python3-pip
export JAVA_HOME=/usr/lib/jvm/jre-17-openjdk
pip3 install ansible ansible-rulebook ansible-runner
```

## Dev Kubernetes cluster using minikube

```shell
$ minikube start \
    --driver=virtualbox \
    --nodes 1 \
    --cni calico \
    --cpus=2 \
    --memory=2g \
    --kubernetes-version=v1.31.0 \
    --container-runtime=containerd \
    --profile k8s-eda

# add ingress if needed
$ minikube addons enable ingress --profile k8s-eda
# or minikube start --addons=ingress

# add /etc/hosts entry for DNS
$ echo "$(minikube ip --profile k8s-eda) k8slab.local" | sudo tee -a /etc/hosts

# housekeeping - delete entries from /etc/hosts once testing done
$ sudo sed -i "/$(minikube ip --profile k8s-eda)/d" /etc/hosts
```


```yaml
- name: Automatic Remediation of a web server
  hosts: all
  sources:
    - name: listen for alerts
      ansible.eda.alertmanager:
        host: 0.0.0.0
        port: 8000
  rules:
    - name: restart web server
      condition: event.alert.labels.job == "fastapi" and event.alert.status == "firing"
      action:
        run_playbook:
          name: ansible.eda.start_app
```


```shell
$ ansible-galaxy collection install sabre1041.eda
```

```shell
ansible-rulebook -i hosts --rulebook rulebooks/k8s-rulebook.yaml --verbose
```

Create a configmap to test

```shell
$ kubectl create configmap -n default eda-example --from-literal=message="Kubernetes Meets Event-Driven Ansible"
configmap/eda-example created
```

## Kubernetes Demo

Create resource

```shell
$ kubectl apply -f kubernetes-yamls/portal-deployment.yaml
namespace/eda-demo created
configmap/video-configmap created
deployment.apps/video created
service/portal-service created
```

Check [http://k8slab.local/portal](http://k8slab.local/portal)

Note: You can also use `kubectl port-forward` to test the same if no ingress is available.

```shell
$ kubectl port-forward service/portal-service 8080:8080 -n eda-demo
Forwarding from 127.0.0.1:8080 -> 5000
Forwarding from [::1]:8080 -> 5000
```

## Troubleshooting


Missing libs

```shell
$  ansible-rulebook --version
...
_libpq
    raise ImportError(
ImportError: no pq wrapper available.
Attempts made:
- couldn't import psycopg 'c' implementation: No module named 'psycopg_c'
- couldn't import psycopg 'binary' implementation: No module named 'psycopg_binary'
- couldn't import psycopg 'python' implementation: libpq library not found
```

```shell
$ sudo dnf install postgresql-devel
```

## References

- [Getting Started with Event-Driven Ansible](https://www.ansible.com/blog/getting-started-with-event-driven-ansible/)
- [github.com/redhat-developer-demos/k8s-ansible-eda](https://github.com/redhat-developer-demos/k8s-ansible-eda)
- [github.com/redhat-developer-demos/ansible-eda-alertmanager](https://github.com/redhat-developer-demos/ansible-eda-alertmanager)
- [OpenShift application monitoring with Event-Driven Ansible & Alertmanager](https://developers.redhat.com/articles/2024/01/08/openshift-application-monitoring-event-driven-ansible-alertmanager#)
- [Event-driven disaster recovery with Red Hat Advanced Cluster Management for Kubernetes and Ansible Automation Platform](https://www.redhat.com/en/blog/event-driven-disaster-recovery)
- [Event-driven Ansible + Gitops](https://www.youtube.com/watch?v=Bb51DftLbPE)(Video)
- [Kubernetes Meets Event-Driven Ansible](https://www.ansible.com/blog/kubernetes-meets-event-driven-ansible/)

Decision Engine image: `registry.redhat.io/ansible-automation-platform-24/de-supported-rhel9:1.0.7-42`