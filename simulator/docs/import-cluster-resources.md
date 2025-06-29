# [Beta] Import your real cluster's resources

There are two ways to import resources from your cluster. These methods cannot be used simultaneously.
- Import resources from your cluster once when initializing the simulator.
- Keep importing resources from your cluster.

## One-shot import: Import resources once when initializing the simulator

To use this, you need to follow these two steps in the simulator configuration:
- Set `true` to `externalImportEnabled`.
- Set the path of the kubeconfig file for your cluster to `KubeConfig`. 
  - This feature only requires the read permission for resources.
- [optional] Set a label selector at `resourceImportLabelSelector` if you want to import specific resources only.

```yaml
externalImportEnabled: true
kubeConfig: "/path/to/your-cluster-kubeconfig"
resourceImportLabelSelector:
  matchLabels:
    env: dev
```

## Syncer: Keep importing resources 

To use this, you need to follow these two steps in the scheduler configuration:
- Set `true` to `resourceSyncEnabled`.
- Set the path of the kubeconfig file for the your cluster to `KubeConfig`. 
  - This feature only requires the read permission for resources.

```yaml
resourceSyncEnabled: true
kubeConfig: "/path/to/your-cluster-kubeconfig"
```

> [!NOTE]
> When you enable `resourceSyncEnabled`, adding/updating/deleting resources directly in the simulator cluster could cause a problem of syncing. 
> You can do them for debugging etc purposes though, make sure you reboot the simulator and the fake source cluster afterward.

### How it syncs Pods

We cannot simply sync all changes to Pods, 
because the real cluster has the scheduler, and it schedules all Pods in the cluster.
If we simply synced all changes to Pods, the scheduling result would also be synced, 
and may conflicted with the decision of another scheduler which is in a fake cluster.

So, we don't sync any of updated events to scheduled Pods.
Pods are synced like:

1. In a real cluster, Pod-a is created
2. In a fake cluster, Pod-a is created. (synced)
3. In a real cluster, the scheduler schedules Pod-a to Node-a. We don't copy this change to a fake cluster.
4. In a fake cluster, the scheduler, which is different one from (3), schedules Pod-a to Node-x.

It means that the scheduling results may be different between a real cluster and a fake cluster. 
But, it's OK.
Our purpose is to create a fake cluster for testing the scheduling, which gets the same load as the production cluster.

### Resources to import

It imports the following resources, which the scheduler's default plugins take into account during scheduling.

- Pods
- Nodes
- PersistentVolumes
- PersistentVolumeClaims
- StorageClasses

If you need to, you can tweak which resources to import via the option in [/simulator/cmd/simulator/simulator.go](https://github.com/kubernetes-sigs/kube-scheduler-simulator/blob/master/simulator/cmd/simulator/simulator.go):

```go
resourceApplierOptions := resourceapplier.Options{
	// GVRsToApply is a list of GroupVersionResource that will be synced.
	// If GVRsToApply is nil, defaultGVRs are used.
	GVRsToApply: []schema.GroupVersionResource{
		{Group: "your-group", Version: "v1", Resource: "your-custom-resources"},
	},

	// Actually, more options are available...

	// FilterBeforeCreating is a list of additional filtering functions that are applied before creating resources.
	FilterBeforeCreating: map[schema.GroupVersionResource][]resourceapplier.FilteringFunction{},
	// MutateBeforeCreating is a list of additional mutating functions that are applied before creating resources.
	MutateBeforeCreating: map[schema.GroupVersionResource][]resourceapplier.MutatingFunction{},
	// FilterBeforeUpdating is a list of additional filtering functions that are applied before updating resources.
	FilterBeforeUpdating: map[schema.GroupVersionResource][]resourceapplier.FilteringFunction{},
	// MutateBeforeUpdating is a list of additional mutating functions that are applied before updating resources.
	MutateBeforeUpdating: map[schema.GroupVersionResource][]resourceapplier.MutatingFunction{},
}
```

> [!NOTE]
> Right now, one-shot import cannot change which resources to import.

## Troubleshooting

### Client Rate Limiter Errors

When importing resources from large clusters, you might encounter "context deadline" or client rate limiter errors. This typically happens when the default QPS (Queries Per Second) and Burst settings are too low to handle the volume of API requests needed for resource synchronization.

**Solution**: If you encounter these errors, consider increasing the QPS and Burst limits in your kubeconfig:

- QPS: Consider setting to 400 (instead of the default 5)
- Burst: Consider setting to 1200 (instead of the default 10)

You can modify these values in your kubeconfig file or client configuration to handle large cluster imports more efficiently. Please note that higher values may increase the load on your API server.