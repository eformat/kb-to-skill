---
name: etcd-health
description: Explores the health of etcd using oc commands. Performs simple remediation for etcd defragmentation. Based on https://access.redhat.com/solutions/5564771
---

## Basic Health Check

Retreive etcd pod names.

```bash
export ETCD_POD_NAME=$(oc get pods -n openshift-etcd -l app=etcd --field-selector="status.phase==Running" -o jsonpath="{.items[0].metadata.name}")
```

List etcd stats table.

```bash
oc exec -n openshift-etcd -c etcd ${ETCD_POD_NAME} -- etcdctl endpoint status --cluster -w json
```

## etcd Leader

The etcd leader can be found by using this check.

```bash
oc exec -n openshift-etcd -c etcd ${ETCD_POD_NAME} -- etcdctl endpoint status --cluster -w json | jq -r '.[] | .Status.header.member_id == .Status.leader'
```

## Check fragmentation ratio

Check that the dbSize and the dbSizeInUse differ in more than 40-50%

```bash
oc exec -n openshift-etcd -c etcd ${ETCD_POD_NAME} -- etcdctl endpoint status --cluster -w json | jq '.[] | ((.Status.dbSize - .Status.dbSizeInUse)/.Status.dbSize)*100'
```

## etcd compact operation

Retrieve the current etcd database versions.

```bash
export CURRENT_VERSIONS=$(oc exec -n openshift-etcd -c etcd ${ETCD_POD_NAME} -- etcdctl endpoint status --write-out json | egrep -o '"revision":[0-9]*' | egrep -o '[0-9]*')
```

Compact the database

```bash
oc exec -n openshift-etcd -c etcd ${ETCD_POD_NAME} -- etcdctl --command-timeout=600s compact $CURRENT_VERSIONS
```

## Additional Notes

Make sure of leaving the leader instance to be the last one to run the command against.

If a timeout occurs increase the --command-timeout until success.

It is important to note that the defrag action is blocking. The member will not respond until the defrag is complete. For this reason, defrag should be a rolling action.
