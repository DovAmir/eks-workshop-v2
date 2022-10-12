---
title: Enabling Fargate
sidebar_position: 10
---

Before you schedule pods on Fargate in your cluster, you must define at least one Fargate profile that specifies which pods use Fargate when launched.

As an administrator, you can use a Fargate profile to declare which pods run on Fargate. You can do this through the profile’s selectors. You can add up to five selectors to each profile. Each selector contains a namespace and optional labels. For every selector, you must define a namespace for every selector. The label field consists of multiple optional key-value pairs. Pods that match a selector are scheduled on Fargate. Pods are matched using a namespace and the labels that are specified in the selector. If a namespace selector is defined without labels, Amazon EKS attempts to schedule all the pods that run in that namespace onto Fargate using the profile. If a to-be-scheduled pod matches any of the selectors in the Fargate profile, then that pod is scheduled on Fargate.

If a pod matches multiple Fargate profiles, you can specify which profile a pod uses by adding the following Kubernetes label to the pod specification: `eks.amazonaws.com/fargate-profile: my-fargate-profile`. The pod must match a selector in that profile to be scheduled onto Fargate. Kubernetes affinity/anti-affinity rules do not apply and aren't necessary with Amazon EKS Fargate pods.

A Fargate profile has been pre-configured in your EKS cluster, and you can inspect it:

```bash
$ aws eks describe-fargate-profile --cluster-name $EKS_CLUSTER_NAME \
  --fargate-profile-name checkout-profile
```

Can you tell what this profile is meant to do?

So why isn't the `checkout` service already running on Fargate? Lets check its labels:

```bash
$ kubectl get pod -n checkout -l app.kubernetes.io/component=service -o json | jq '.items[0].metadata.labels'
```

Looks like our pod is missing the label `fargate=yes`, so lets fix that by updating the deployment for that service so the Pod spec includes the label needed for the profile to schedule it on Fargate.

```kustomization
fundamentals/fargate/enabling/deployment.yaml
Deployment/checkout
```

Apply the kustomization to the cluster:

```bash
$ kubectl apply -k /workspace/modules/fundamentals/fargate/enabling
[...]
$ kubectl rollout status -n checkout deployment/checkout
```

This will cause the pod specification for the `checkout` service to be updated and trigger a new deployment, replacing all the pods. When the new pods are scheduled, the Fargate scheduler will match the new label applied by the kustomization with our target profile and intervene to ensure our pod is schedule on capacity managed by Fargate.


How can we confirm that it worked? Describe the new pod thats been created and take a look at the `Events` section:

```bash
$ kubectl describe pod -n checkout -l app.kubernetes.io/component=service
[...]
Events:
  Type     Reason           Age    From               Message
  ----     ------           ----   ----               -------
  Warning  LoggingDisabled  10m    fargate-scheduler  Disabled logging because aws-logging configmap was not found. configmap "aws-logging" not found
  Normal   Scheduled        9m48s  fargate-scheduler  Successfully assigned checkout/checkout-78fbb666b-fftl5 to fargate-ip-10-42-11-96.us-west-2.compute.internal
  Normal   Pulling          9m48s  kubelet            Pulling image "watchn/watchn-checkout:build.1615751790"
  Normal   Pulled           9m5s   kubelet            Successfully pulled image "watchn/watchn-checkout:build.1615751790" in 43.258137629s
  Normal   Created          9m5s   kubelet            Created container checkout
  Normal   Started          9m4s   kubelet            Started container checkout
```

The events from `fargate-scheduler` give us some insight in to what has happened. The entry we are mainly interested in at this stage in the lab is the event with the reason `Scheduled`. Inspecting that closely gives us the name of the Fargate instance that was provisioned for this pod, in the case of the above example this is `fargate-ip-10-42-11-96.us-west-2.compute.internal`.

We can inspect this node from `kubectl` to get additional information about the compute that was provisioned for this pod:

```bash
$ NODE_NAME=$(kubectl get pod -n checkout -l app.kubernetes.io/component=service -o json | jq -r '.items[0].spec.nodeName')
$ kubectl describe node $NODE_NAME
Name:               fargate-ip-10-42-11-96.us-west-2.compute.internal
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    eks.amazonaws.com/compute-type=fargate
                    failure-domain.beta.kubernetes.io/region=us-west-2
                    failure-domain.beta.kubernetes.io/zone=us-west-2b
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=ip-10-42-11-96.us-west-2.compute.internal
                    kubernetes.io/os=linux
                    topology.kubernetes.io/region=us-west-2
                    topology.kubernetes.io/zone=us-west-2b
[...]
```

This provides us with a number of insights in to the nature of the underlying compute instance:
- The label `eks.amazonaws.com/compute-type` confirms that a Fargate instance was provisioned
- Another label `topology.kubernetes.io/zone` specified the availability zone that the pod is running in
- In the `System Info` section (not shown above) we can see that the instance is running Amazon Linux 2, as well as the version information for system components like `container`, `kubelet` and `kube-proxy`