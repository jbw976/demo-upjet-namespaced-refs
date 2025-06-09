# Namespaced Cross Resource References in Crossplane v2

## Prerequisites
```
kind create cluster

helm repo add crossplane-master https://charts.crossplane.io/master/
helm repo update
helm install crossplane --namespace crossplane-system --create-namespace crossplane-master/crossplane --devel

kubectl create namespace ns-ref-test
```

### Install and Configure v2 Compatible Provider and Functions
```
kubectl apply -f provider.yaml
kubectl apply -f functions.yaml
kubectl get pkg
```

### Configure AWS Credentials
```
AWS_PROFILE=default && echo -e "[default]\naws_access_key_id = $(aws configure get aws_access_key_id --profile $AWS_PROFILE)\naws_secret_access_key = $(aws configure get aws_secret_access_key --profile $AWS_PROFILE)" > aws-credentials.txt
kubectl create secret generic aws-secret -n crossplane-system --from-file=creds=./aws-credentials.txt
kubectl apply -f providerconfig.yaml
```

## 1. Create S3 Bucket directly using namespaced Managed Resources

This example creates an S3 bucket, along with the necessary configuration to
create private bucket ACLs. S3 discourages the use of ACLs nowadays, so some
[extra configuration is
needed](https://www.learnaws.org/2023/08/26/aws-s3-bucket-does-not-allow-acls/),
such as ownership controls and public access block configuration.

These resources are all namespaced and use cross resource references to resolve
their references and refer to the bucket.
```
kubectl create -f managed-resources/bucket.yaml
kubectl apply -f managed-resources/bucket-ownership-controls.yaml
kubectl apply -f managed-resources/bucket-public-access-block.yaml
kubectl apply -f managed-resources/bucket-acl.yaml
```

Ensure that the Bucket and its related resources get created, ready, and synced:
```
kubectl get managed -A

NAMESPACE     NAME                                                  SYNCED   READY   EXTERNAL-NAME                     AGE
ns-ref-test   bucketacl.s3.aws.m.upbound.io/crossplane-bucket-acl   True     True    crossplane-bucket-6cwhq,private   57s

NAMESPACE     NAME                                                                               SYNCED   READY   EXTERNAL-NAME             AGE
ns-ref-test   bucketownershipcontrols.s3.aws.m.upbound.io/crossplane-bucket-ownership-controls   True     True    crossplane-bucket-6cwhq   57s

NAMESPACE     NAME                                                                                SYNCED   READY   EXTERNAL-NAME             AGE
ns-ref-test   bucketpublicaccessblock.s3.aws.m.upbound.io/crossplane-bucket-public-access-block   True     True    crossplane-bucket-6cwhq   57s

NAMESPACE     NAME                                                 SYNCED   READY   EXTERNAL-NAME             AGE
ns-ref-test   bucket.s3.aws.m.upbound.io/crossplane-bucket-6cwhq   True     True    crossplane-bucket-6cwhq   57s
```

We can also see that these namespaced resources, like `BucketACL`, resolved
their references successfully and now directly refer to their bucket:
```
kubectl -n ns-ref-test get bucketacl.s3.aws.m.upbound.io/crossplane-bucket-acl -o json | jq .spec.forProvider.bucketRef
{
  "name": "crossplane-bucket-6cwhq"
}
```

## 2. Create S3 Bucket using a namespaced XR

Now let's create similar namespaced `Bucket` resources using composition, to
ensure namespaced cross resource references work in that context as well.

Create the `CompositeResourceDefinition` (XRD) and `Composition`:
```
kubectl apply -f composition/xrd.yaml
kubectl apply -f composition/composition.yaml
```

Create a namespaced `PrivateBucket` XR instance:
```
kubectl apply -f composition/xr.yaml
```

The namespaced `Bucket` resources should become ready and available:
```
kubectl tree -n ns-ref-test privatebucket.demo.crossplane.io/private-bucket

NAMESPACE    NAME                                                          READY  REASON     AGE
ns-ref-test  PrivateBucket/private-bucket                                  True   Available  3m11s
ns-ref-test  ├─Bucket/private-bucket-jwqdq                                 True   Available  3m11s
ns-ref-test  ├─BucketACL/private-bucket-acl                                True   Available  3m11s
ns-ref-test  ├─BucketOwnershipControls/private-bucket-ownership-controls   True   Available  3m11s
ns-ref-test  └─BucketPublicAccessBlock/private-bucket-public-access-block  True   Available  3m11s
```

The namespaced references should be resolved as well, for example:
```
kubectl -n ns-ref-test get bucketacl.s3.aws.m.upbound.io/private-bucket-acl -o json | jq .spec.forProvider.bucketRef
{
  "name": "private-bucket-jwqdq"
}
```

## 3. Create EC2 Instance and Networking using Multiple Resolution

We can also create a namespaced EC2 instance with its required networking, like VPC,
Subnet, and Security Groups. The security groups will be selected for the EC2
instance using multiple reference resolution, i.e. one reference selector can
select multiple objects.

Create the `CompositeResourceDefinition` (XRD) and `Composition`:
```
kubectl apply -f multi-refs/xrd.yaml
kubectl apply -f multi-refs/composition.yaml
```

Create a namespaced `ComputeInstance`:
```
kubectl apply -f multi-refs/xr.yaml
```

The namespaced `ComputeInstance` resources should become ready and available:
```
kubectl tree -n ns-ref-test computeinstance.demo.crossplane.io/cool-compute

NAMESPACE    NAME                               READY  REASON     AGE
ns-ref-test  ComputeInstance/cool-compute       True   Available  4m21s
ns-ref-test  ├─Instance/cool-compute-instance   True   Available  4m21s
ns-ref-test  ├─SecurityGroup/cool-compute-sg-0  True   Available  4m21s
ns-ref-test  ├─SecurityGroup/cool-compute-sg-1  True   Available  4m21s
ns-ref-test  ├─SecurityGroup/cool-compute-sg-2  True   Available  4m21s
ns-ref-test  ├─Subnet/cool-compute-subnet       True   Available  4m21s
ns-ref-test  └─VPC/cool-compute-vpc             True   Available  4m21s
```

The namespaced reference using multiple reference resolution should be resolved:
```
kubectl -n ns-ref-test get instance.ec2.aws.m.upbound.io/cool-compute-instance -o json | jq .spec.forProvider.vpcSecurityGroupIds
[
  "sg-065d79d3f6fa6a851",
  "sg-06835b7e42db26c26",
  "sg-08e47b350f0c006c5"
]
```

## 4. Create Legacy Cluster Scoped Resources

Create the legacy cluster scoped `Bucket` and all of its resources:
```
kubectl create -f legacy/bucket.yaml
kubectl apply -f legacy/bucket-ownership-controls.yaml
kubectl apply -f legacy/bucket-public-access-block.yaml
kubectl apply -f legacy/bucket-acl.yaml
```

Ensure that the legacy Bucket and its related resources get created, ready, and
synced (note they have no namespaced, they are cluster scoped):
```
kubectl get managed -A

NAMESPACE   NAME                                                       SYNCED   READY   EXTERNAL-NAME                     AGE
            bucketacl.s3.aws.upbound.io/crossplane-bucket-acl-legacy   True     True    crossplane-bucket-cqcfz,private   2m

NAMESPACE   NAME                                                                                    SYNCED   READY   EXTERNAL-NAME             AGE
            bucketownershipcontrols.s3.aws.upbound.io/crossplane-bucket-ownership-controls-legacy   True     True    crossplane-bucket-cqcfz   78s

NAMESPACE   NAME                                               SYNCED   READY   EXTERNAL-NAME             AGE
            bucket.s3.aws.upbound.io/crossplane-bucket-cqcfz   True     True    crossplane-bucket-cqcfz   2m1s

NAMESPACE   NAME                                                                                     SYNCED   READY   EXTERNAL-NAME             AGE
            bucketpublicaccessblock.s3.aws.upbound.io/crossplane-bucket-public-access-block-legacy   True     True    crossplane-bucket-cqcfz   2m1s
```

We can also see that these legacy resources, like `BucketACL`, resolved
their references successfully and now directly refer to their bucket:
```
kubectl get bucketacl.s3.aws.upbound.io/crossplane-bucket-acl-legacy -o json | jq .spec.forProvider.bucketRef
{
  "name": "crossplane-bucket-cqcfz"
}
```

## Clean-up

Clean up all the resources, if there are any hangs, make sure the bucket is
deleted and the rest should resolve themselves:
```
kubectl delete -f managed-resources/bucket-ownership-controls.yaml
kubectl delete -f managed-resources/bucket-public-access-block.yaml
kubectl delete -n ns-ref-test bucket.s3.aws.m.upbound.io -l demo.crossplane.io/scenario=namespaced-refs-bucket
kubectl delete -f managed-resources/bucket-acl.yaml
```

```
kubectl delete -f legacy/bucket-ownership-controls.yaml
kubectl delete -f legacy/bucket-public-access-block.yaml
kubectl delete bucket.s3.aws.upbound.io -l demo.crossplane.io/scenario=legacy-refs-bucket
kubectl delete -f legacy/bucket-acl.yaml
```

```
kubectl delete -f composition/xr.yaml
kubectl delete -f multi-refs/xr.yaml
```

Ensure no more resources are left behind:
```
kubectl get managed -A
```