# CA証明書のプロビジョニングとTLS証明書の生成

本実習では、CloudFlareのPKIツールキットである[cfssl](https://github.com/cloudflare/cfssl)を使用して[公開鍵基盤](https://ja.wikipedia.org/wiki/%E5%85%AC%E9%96%8B%E9%8D%B5%E5%9F%BA%E7%9B%A4)をプロビジョニングし、それを使用して認証機関を起動し、etcd、kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxyの各コンポーネントのTLS証明書を生成します。

## Certificate Authority

本セクションでは、追加のTLS証明書を生成するために使用できる認証局をプロビジョニングします。

CA構成ファイル、証明書、および秘密鍵を生成します:

```sh
{

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```

結果:

```
ca-key.pem
ca.pem
```

## クライアント及びサーバー証明書

本セクションでは、各Kubernetesコンポーネントのクライアント証明書とサーバー証明書、およびKubernetes管理ユーザーのクライアント証明書を生成します。

### admin用クライアント証明書

`admin`クライアント証明書と秘密鍵を生成します:

```sh
{

cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

}
```

結果:

```
admin-key.pem
admin.pem
```

### Kubeletクライアント証明書

KubernetesはNode Authorizerと呼ばれる[専用の許可モード](https://kubernetes.io/docs/admin/authorization/node/)を使って、Kubeletが行うAPIリクエストを許可します。KubeletがNode Authorizerによる認証を受けるには、`system:nodes`グループに属していることをユーザ名`system:node:<ノード名>`で識別するクレデンシャルを使用する必要があります。本セクションでは、各Kubernetesワーカーノード用にNode Authorizerの要件を満たす証明書を作成します。

各ワーカーノード用の証明書と秘密鍵を生成します:

```sh
for instance in worker-{0..2}; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

EXTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')

INTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].networkIP)')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

結果:

```
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem
```

### Controller Manager用クライアント証明書

`kube-controller-manager`クライアント証明書と秘密鍵を生成します:

```sh
{

cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```

結果:

```
kube-controller-manager-key.pem
kube-controller-manager.pem
```


### Kube Proxy用クライアント証明書

`kube-proxy`クライアント証明書と秘密鍵を生成します:

```sh
{

cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```

結果:

```
kube-proxy-key.pem
kube-proxy.pem
```

### Scheduler用クライアント証明書

`kube-scheduler`クライアント証明書と秘密鍵を生成します:

```sh
{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
```

結果:

```
kube-scheduler-key.pem
kube-scheduler.pem
```


### KubernetesのAPIサーバー用証明書

静的IPアドレス`kubernetes-the-hard-way`はKubernetesのAPIサーバー用証明書のSubject Alternative Name(SAN)リストに含まれます。これによって証明書をリモートクライアントで検証できるようになります。

KubernetesのAPIサーバー用クライアント証明書と秘密鍵を生成します:

```sh
{

KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')

KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```

> KubernetesのAPIサーバーには自動的にKubernetesの内部DNS名が割り当てられます。この名前は、[コントロールプレーンのブートストラップ](08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-api-server)で内部クラスタサービス用に予約されたアドレス範囲(`10.32.0.0/24`)の最初のIPアドレス(`10.32.0.1`)にリンクされます。

結果:

```
kubernetes-key.pem
kubernetes.pem
```

## サービスアカウントのキーペア

Kubernetesのコントローラーマネージャーは、[サービスアカウントの管理](https://kubernetes.io/docs/admin/service-accounts-admin/)に関するドキュメントで説明されているように、キーペアを使用してサービスアカウントトークンを生成して署名します。

`service-account`用クライアント証明書と秘密鍵を生成します:

```sh
{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}
```

結果:

```
service-account-key.pem
service-account.pem
```


## クライアント及びサーバー用証明書の配布

適切な証明書と秘密鍵を各ワーカーノード用インスタンスにコピーします:

```sh
for instance in worker-{0..2}; do
  gcloud compute scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:~/
done
```

適切な証明書と秘密鍵を各コントロールプレーン用インスタンスにコピーします:

```sh
for instance in controller-{0..2}; do
  gcloud compute scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ${instance}:~/
done
```

>`kube-proxy`、`kube-controller-manager`、`kube-scheduler`、および`kubelet`用クライアント証明書は、次の実習でクライアントの認証用設定ファイルを生成するために使用します。

Next: [認証用Kubernetes設定ファイルの生成](05-kubernetes-configuration-files.md)
