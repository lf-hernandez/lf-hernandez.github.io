---
title: "Getting a Service on EKS access to Bedrock"
date: 2026-06-28T11:00:00-04:00
summary: "How to wire up a service running on EKS to call Amazon Bedrock."
tags: ["kubernetes", "aws", "eks", "bedrock"]
categories: ["kubernetes", "aws"]
---
Recently, I needed to get a service running on an EKS cluster to talk to Amazon Bedrock. The code was in place, I just needed to figure out how the service would reach out to Bedrock. The pod had no AWS credentials, so I knew I had to set up authentication... I just wasn't sure which mechanism.

Normally, you'd solve this using IAM Roles for Service Accounts (IRSA). You'd annotate the service account, stand up an OpenID Connect (OIDC) provider, maintain a per-cluster trust policy. But as I dug a little into what options were available, Pod Identity stuck out as a more straightforward approach. I went with EKS Pod Identity, which does the same job as IRSA with a lot less ceremony. Here's the path that worked for me.

In simple terms, both IRSA and Pod Identity let a pod assume an IAM role, the difference is the plumbing. IRSA depends on an OIDC provider and a trust policy you wire up per cluster, while Pod Identity hands that off to an EKS-managed agent so you skip the OIDC setup.

**1. Install the agent**:

```bash
aws eks create-addon --cluster-name my-cluster --addon-name eks-pod-identity-agent
```

**2. Create an IAM role** whose trust policy allows the EKS service principal. This is the same generic trust for every Pod Identity role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "pods.eks.amazonaws.com" },
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }
  ]
}
```

Then attach a permissions policy for Bedrock (`bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream`).

**3. Create the service account** and associate the role with it:

```bash
kubectl create serviceaccount bedrock-caller -n my-app

aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace my-app \
  --service-account bedrock-caller \
  --role-arn arn:aws:iam::111122223333:role/bedrock-pod-identity
```

**4. Use that service account** in the pod (`serviceAccountName: bedrock-caller`). No need for an annotation on the service account; the association lives in the EKS API. The agent injects credentials and the AWS SDK picks them up automatically... like magic.

One gotcha that is easy to miss, make sure that the model is actually enabled in the Bedrock console. Model access is a separate toggle from IAM permissions.

And Bob's your uncle, I hope this was helpful and definitely give the docs a read and dive deeper!

---

Are you leveling up on Kubernetes, Linux, or cloud-native skills? You can use my code **`LFHERNANDEZ`** for a discount on the [full catalog](https://training.linuxfoundation.org/full-catalog/) of training courses and certifications (CKA, CKAD, CKS, LFCS, and many many more).

Docs: [EKS Pod Identities](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
