# Troubleshooting

This document contains information about various issues you might face
and how to solve them.

## ErrImagePull due to missing authentication

The deployed containers pull the images from private containers registries that
can potentially return authentication errors like:

```
Failed to pull image "registry.redhat.io/rhosp-rhel9/openstack-rabbitmq:17.0":
rpc error: code = Unknown desc = unable to retrieve auth token: invalid
username/password: unauthorized: Please login to the Red Hat Registry using
your Customer Portal credentials.
```

An example of a failed pod:

```
  Normal   Scheduled       3m40s                  default-scheduler  Successfully assigned openstack/rabbitmq-server-0 to worker0
  Normal   AddedInterface  3m38s                  multus             Add eth0 [10.101.0.41/23] from ovn-kubernetes
  Warning  Failed          2m16s (x6 over 3m38s)  kubelet            Error: ImagePullBackOff
  Normal   Pulling         2m5s (x4 over 3m38s)   kubelet            Pulling image "registry.redhat.io/rhosp-rhel9/openstack-rabbitmq:17.0"
  Warning  Failed          2m5s (x4 over 3m38s)   kubelet            Failed to pull image "registry.redhat.io/rhosp-rhel9/openstack-rabbitmq:17.0": rpc error: code  ... can be found here: https://access.redhat.com/RegistryAuthentication
  Warning  Failed          2m5s (x4 over 3m38s)   kubelet            Error: ErrImagePull
  Normal   BackOff         110s (x7 over 3m38s)   kubelet            Back-off pulling image "registry.redhat.io/rhosp-rhel9/openstack-rabbitmq:17.0"
```

To solve this issue we need to get a valid pull-secret from the official [Red
Hat console site](https://console.redhat.com/openshift/install/pull-secret),
store this pull secret locally in a machine with access to the Kubernetes API
(service node), and then run:

```
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=<pull_secret_location.json>
```

The previous command will make available the authentication information in all
the cluster's compute nodes, then trigger a new pod deployment to pull the
container image with:

```
kubectl delete pod rabbitmq-server-0 -n openstack
```

And the pod should be able to pull the image successfully.  For more
information about what container registries requires what type of
authentication, check the [official
docs](https://access.redhat.com/RegistryAuthentication).

## Database schema version mismatch when restoring OVN databases

You may see the following errors when trying to restore OVN databases:

```
ovsdb-client: backup schema has version "7.0.0" but database schema has version "6.3.0" (use --force to override differences, or "ovsdb-client convert" to change the schema)
ovsdb-client: backup schema has version "20.27.0" but database schema has version "20.23.0" (use --force to override differences, or "ovsdb-client convert" to change the schema)
```

This happens because your podified cluster is running OVN version that is older
than the version running in the original cluster. You may work this issue
around by pulling OVN images from the original cluster registry into podified
cluster.

You may have to first enable insecure registries in your podified cluster:

```bash
oc patch image.config.openshift.io/cluster --type=merge --patch '
spec:
  registrySources:
    insecureRegistries:
    - 192.168.130.1
'
```

Then update OVN services to pull images from the original registry:

```bash
oc patch openstackcontrolplane openstack --type=merge --patch '
spec:
  ovn:
    template:
      ovnDBCluster:
        ovndbcluster-nb:
          containerImage: 192.168.130.1:8787/rh-osbs/rhosp17-openstack-ovn-nb-db-server:17.1_20230130.1
        ovndbcluster-sb:
          containerImage: 192.168.130.1:8787/rh-osbs/rhosp17-openstack-ovn-sb-db-server:17.1_20230130.1
      ovnNorthd:
        containerImage: 192.168.130.1:8787/rh-osbs/rhosp17-openstack-ovn-nb-db-server:17.1_20230130.1
'
```

Now repeat database `ovsdb-client restore` commands. They should succeed.
