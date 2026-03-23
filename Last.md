
| **Component**             | **Critical Troubleshooting Fields**                                                                       |     |     |     |     |
| ------------------------- | --------------------------------------------------------------------------------------------------------- | --- | --- | --- | --- |
| **`kube-apiserver.yaml`** | `--etcd-servers`, `--client-ca-file`, `--tls-cert-file`, `--service-cluster-ip-range`                     |     |     |     |     |
| **`etcd.yaml`**           | `--data-dir`, `--listen-client-urls`, `--advertise-client-urls`, cert paths (`--cert-file`, `--key-file`) |     |     |     |     |
| **`kubelet`**             | Located at `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`. Check for the `--config` path.        |     |     |     |     |
|                           |                                                                                                           |     |     |     |     |
|                           |                                                                                                           |     |     |     |     |
