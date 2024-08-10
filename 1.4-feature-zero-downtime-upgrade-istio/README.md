## Lab 21 - Zero Downtime Istio Upgrades <a name="lab-21---zero-downtime-istio-upgrades-"></a>


![Upgrade Istio](images/upgrade-istio.png)
Historically, upgrading Istio without downtime has been a complicated ordeal. Today, Gloo Platform can easily facilitate zero downtime upgrades using its Istio Lifecycle Manager. Gloo Platform allows you to deploy multiple versions if Istio side-by-side and transition from one version to the other. 
This lab will show you how to upgrade Istio quickly and safely without impacting client traffic. This is done through the use of Istio revisions. 

Links:
- [Istio Production Best Practices](https://docs.solo.io/gloo-mesh-enterprise/latest/setup/prod/manual/namespaces/)
- [Gloo Platform Managed Istio](https://docs.solo.io/gloo-mesh-enterprise/latest/setup/installation/istio/gm_managed_istio/)
- [GatewayLifecycleManager API](https://docs.solo.io/gloo-mesh-enterprise/latest/reference/api/gateway_lifecycle_manager/)
- [IstioLifecycleManager API](https://docs.solo.io/gloo-mesh-enterprise/latest/reference/api/istio_lifecycle_manager/)
## Upgrade Istio using Helm in Cluster: remote-1

* To upgrade Istio we will deploy a whole new canary version beside it. We will also deploy new gateways and migrate traffic to them.

```shell
helm upgrade --install istio-base istio/base \
  -n istio-system \
  --kube-context=r1 \
  --version 1.22.0 \
  --set defaultRevision=1-22

helm upgrade -i istiod-1-22 istio/istiod \
  --set revision=1-22 \
  --version 1.22.0 \
  --namespace istio-system  \
  --kube-context=r1 \
  --set "global.multiCluster.clusterName=remote-1" \
  --set "meshConfig.trustDomain=remote-1" \
  -f data/istiod-values.yaml
```

* In this lab we will show you two options for upgrading gateways, in-place and via revisions. Depending on your needs and requirements you can choose the best option that will work for you.

* Upgrade the Istio eastwest gateway in place
```shell
helm upgrade -i istio-eastwestgateway istio/gateway \
  --set revision=1-22 \
  --version 1.22.0 \
  --namespace istio-gateways  \
  --kube-context=r1 \
  -f data/eastwest-values.yaml
```

* Deploy new Istio 1-22 ingress gateway using revisions
```shell
helm upgrade -i istio-ingressgateway-1-22 istio/gateway \
  --set revision=1-22 \
  --version 1.22.0 \
  --namespace istio-gateways  \
  --kube-context=r1 \
  -f data/ingress-values.yaml
```

* Update the standalone Kubernetes service send traffic to the new Istio ingressgateway.
```shell
kubectl apply --context r1 -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  namespace: istio-gateways
  labels:
    app: gloo-gateway
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "instance"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    istio: ingressgateway-1-22
  ports:
  # Port for health checks on path /healthz/ready.
  # For AWS ELBs, this port must be listed first.
  - name: status-port
    port: 15021
    targetPort: 15021
  # main http ingress port
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http2
  # main https ingress port
  - port: 443
    protocol: TCP
    targetPort: 443
    name: https
EOF
```

* Remove Istio 
```shell
helm uninstall istio-ingressgateway-1-21 \
  --namespace istio-gateways  \
  --kube-context=r1

helm uninstall istiod-1-21 \
  --namespace istio-system  \
  --kube-context=r1
```

* Verify only Istio 1-22 is running
```shell
istioctl proxy-status --context r1
```

## Upgrade Istio using Helm in Cluster: remote-2

* Upgrade Istiod to 1-22 components
```shell
helm upgrade --install istio-base istio/base \
  --kube-context=r2 \
  -n istio-system \
  --version 1.22.0 \
  --set defaultRevision=1-22

helm upgrade -i istiod-1-22 istio/istiod \
  --set revision=1-22 \
  --version 1.22.0 \
  --namespace istio-system  \
  --kube-context=r2 \
  --set "global.multiCluster.clusterName=remote-2" \
  --set "meshConfig.trustDomain=remote-2" \
  -f data/istiod-values.yaml

```

* Upgrade the Istio eastwest gateway in place
```shell
helm upgrade -i istio-eastwestgateway istio/gateway \
  --set revision=1-22 \
  --version 1.22.0 \
  --namespace istio-gateways  \
  --kube-context=r2 \
  -f data/eastwest-values.yaml
```

* Deploy new Istio 1-22 ingress gateway using revisions
```shell
helm upgrade -i istio-ingressgateway-1-22 istio/gateway \
  --set revision=1-22 \
  --version 1.22.0 \
  --namespace istio-gateways  \
  --kube-context=r2 \
  -f data/ingress-values.yaml
```

* Update the standalone Kubernetes service send traffic to the new Istio ingressgateway.
```shell
kubectl apply --context r2 -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  namespace: istio-gateways
  labels:
    app: gloo-gateway
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "instance"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    istio: ingressgateway-1-22
  ports:
  # Port for health checks on path /healthz/ready.
  # For AWS ELBs, this port must be listed first.
  - name: status-port
    port: 15021
    targetPort: 15021
  # main http ingress port
  - port: 80
    protocol: TCP
    targetPort: 80
    name: http2
  # main https ingress port
  - port: 443
    protocol: TCP
    targetPort: 443
    name: https
EOF
```

* Remove Istio 
```shell
helm uninstall istiod-1-21 \
  --namespace istio-system  \
  --kube-context=r2
```

* Verify only Istio 1-22 is running
```shell
istioctl proxy-status --context r2
```


