# Mount Volumes

This page shows how to add volumes to your Spark applications.

## Standard configuration

By default, Spark uses temporary storage to spill data to disk during shuffles and other operations when no volume is present. With Spark on Kubernetes, pods are initialized with an emptyDir volume for each directory listed in spark.local.dir. When the pod is deleted, the ephemeral storage will be cleared and the data removed.

For certain Spark applications with larger shuffle workfloads or a need to persist data beyond the life of the application, it may be desirable to mount an external volume. Depending on your cloud provider, there are several volume options you can choose from to make your Spark workspaces more dynamic and scalable.

By default, if you select a node type that includes scalable block storage, such as d instances on AWS (m5d, r5d, c5d, etc), Ocean Spark will automatically mount this volume and configure Spark to operate within it.

## Mount secrets as files in volumes

Instead of setting environment variables from [Kubernetes secrets](ocean-spark/configure-spark-apps/secrets-environment-variables), you can also mount secrets directly as files into a volume.

First, create a kubernetes secret in your cluster, and make sure to use the namespace spark-apps so your application can access it. Here is an example basic-auth secret below:

```yaml
apiVersion: v1
kind: Secret
metadata:
  namespace: spark-apps
  name: basic-auth
data:
  password: cGFzc3dvcmQK # password
  user: dXNlcgo= # user
type: Opaque
```

In your application json config, add a volume object that references the secret name, data key you would like to include in the volume, and a name for the volume reference. In your executor or driver config, add a volumeMounts object that includes the volume name referenced in the previous step and path where you would like the file to be mounted. You can find an example application json config below that creates a volume named volume-secret that references the user key in the basic-auth secret and mounts that secret file to /opt/spark/work-dir/secrets on the executor pod(s).

```json
{
  "executor": {
    "cores": 1,
    "instances": 3,
    "instanceType": "r5.xlarge",
    "volumeMounts": [
      {
        "mountPath": "/opt/spark/work-dir/secrets",
        "name": "volume-secret"
      }
    ]
  },
  "volumes": [
    {
      "secret": {
        "items": [
          {
            "key": "user",
            "path": "secret.yaml"
          }
        ],
        "secretName": "basic-auth"
      },
      "name": "volume-secret"
    }
  ]
}
```

If you `kubectl exec` into one of the executor pods, you will find the `secrets.yaml` file located at `/opt/spark/work-dir/secrets`.

## Mount persistent cloud volumes to your container

For larger volume needs, sharing volumes across applications, or persisting data beyond an application's lifecycle, you can mount a cloud provider volume to your container. Instead of passing in a mount path, find the appropriate key for the cloud volume of your choice, and add the necessary data to the object. You can find the full list of supported volume options in the [API reference](https://docs.spot.io/api/#operation/OceanSparkClusterApplicationSubmit) for submitting applications.

To enable volume mounting, you must first create the volume instance in your respective cloud provider. Then, you must make sure your cluster role has read/write access to the volume.

On AWS, you can create an AWS Elastic Block Storage in the AWS console, or by running the following command:

```
aws ec2 create-volume --availability-zone <your availability zone> <name of EBS>
```

You can see more configuration options in the [EBS documentation](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-volume.html).

To mount an EBS volume to your spark application, add the following json to your application submission:

```json
{
  "volumes": [
    {
      "name": "spark-aws-dir",
      "awsElasticBlockStore": {
        "fsType": "type of file system",
        "readOnly": false,
        "volumeID": "id of ebs"
      }
    }
  ]
}
```

## Dynamically provision volumes using Persistent Volume Claims

As of Spark 3.1, you can dynamically provision and mount volumes to both your executor and driver pods using persistent volume claims. Persistent volume claims are kubernetes resources that allow you to allocate and mount resources of an elastic disk to your kubernetes pods. When your application is complete, kubernetes will remove the pod and delete the volume from your cloud provider. You can read more about the different configuration options in [Running Spark on Kubernetes](https://spark.apache.org/docs/latest/running-on-kubernetes.html#using-kubernetes-volumes).

To enable PVC provisioning, you must add a few configuration settings to your sparkConf:

```json
{
  "sparkConf": {
    "spark.kubernetes.executor.volumes.persistentVolumeClaim.data.options.claimName": "OnDemand",
    "spark.kubernetes.executor.volumes.persistentVolumeClaim.data.options.storageClass": "standard",
    "spark.kubernetes.executor.volumes.persistentVolumeClaim.data.options.sizeLimit": "500Gi",
    "spark.kubernetes.executor.volumes.persistentVolumeClaim.data.mount.path": "/var/data",
    "spark.kubernetes.executor.volumes.persistentVolumeClaim.data.mount.readOnly": "false"
  }
}
```

In the sample above, the claimName must be set to OnDemand. You can set the mount path to anywhere you like on the container, just make sure spark is aware of the drive and actually using it. You may need to alter the setting spark.local.dir to the location of your mount point.

In addition, you must also attach a cluster role of edit to the spark-driver service account. You can run the following command to attach the cluster role:

```
kubectl create clusterrolebinding <cluster-role-binding-name> --clusterrole=edit --serviceaccount=spark-apps:spark-driver --namespace=default
```

## What’s Next?

Take the [Product Tour](ocean-spark/product-tour/) and learn about everything you can do in the Ocean Spark console.
