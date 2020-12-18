# Argo Rollout official Demo

```
git clone https://github.com/argoproj/rollouts-demo
```

## Install istio

```
../../setup/istio/install.sh
```

## Canary demo with Istio

- Apply the manifests of the canary example:

```
kustomize build examples/istio | kubectl apply -f -
```

Look are the rollout definition :

```
kubectl get rollout istio-rollout -o yaml -n rollouts-demo-istio
```

It is very close to a Deployment definition, but with an additionnal `strategy:canary` section.

- Watch the rollout or experiment using the argo rollouts kubectl plugin:

```
kubectl argo rollouts get rollout istio-rollout --watch
```

- Trigger an update by setting the image of a new color to run:

```
kubectl argo rollouts set image istio-rollout "*=argoproj/rollouts-demo:yellow"
```

Look at the rollout object you are currently watching : a canary version (new replicaset) has been created, and 20% of the traffic is routed to it.
Then, the rollout is in "paused" as asked in the rollout YAML definition.
So we have to promote it to continue:

```
kubectl argo rollouts promote istio-rollout
```

Look at the rollout : rollout progress to step 2 by increasing the percentage of requests routed to the canary to 40, 60, 80.
To answer to the increasing demand, more pods are created in the canary replicaset, until all the traffic is routed to the canary version.