# Using StatefulSet

- In this example, the StatefulSet creates 4 Pods.
- Altibase Kubernetes Utility supports up to 4 pods.

##### Write a StatefulSet yaml file

```yaml
# file : altibase-sts.yaml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: altibase-sts
spec:
  serviceName: altibase-svc
  replicas: 4
  podManagementPolicy: OrderedReady
  selector:
    matchLabels:
      app: altibase-sts
  template:
    metadata:
      labels:
        app: altibase-sts
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: altibase-sts
        image: altibase/7.1-bare
        command:
        - /bin/bash
        - "-c"
        - /home/altibase/config_map/entry_point.sh
        ports:
        - containerPort: 20300
          protocol: TCP
        - containerPort: 20301
          protocol: TCP
        resources:
          requests:
            cpu: 250m
          limits:
            cpu: 3 
        startupProbe:
          exec:
            command:
            - cat
            - /tmp/aku_start_completed
          failureThreshold: 3600
          periodSeconds: 10
        volumeMounts:
        - name: altibase-pv
          mountPath: /ALTIBASE
        - name: altibase-cm
          mountPath: /home/altibase/config_map
          readOnly: true
      volumes:
        - name: altibase-cm
          configMap:
            name: altibase-cm
            defaultMode: 0777
            items:
            - key: "license"
              path: "license"
            - key: "set_altibase.env"
              path: "set_altibase.env"
            - key: "aku.conf"
              path: "aku.conf"
            - key: "sample_schema.sql"
              path: "sample_schema.sql"
            - key: "entry_point.sh"
              path: "entry_point.sh"
  volumeClaimTemplates:
    - metadata:
        name: altibase-pv
      spec:
        accessModes: [ "ReadWriteMany" ]
        resources:
          requests:
            storage: 10Gi
```

##### Create StatefulSet

```
$ kubectl create -f altibase-sts.yaml
statefulset.apps/altibase-sts created
```

##### Confirm StatefulSet and Pod creation

```
$ kubectl get sts -o wide
NAME                 READY   AGE    CONTAINERS               IMAGES
altibase-sts         4/4     12m    altibase-sts             altibase/7.1-bare

$ kubectl get pod -o wide
NAME                   READY   STATUS    RESTARTS         AGE    IP             NODE      NOMINATED NODE   READINESS GATES
altibase-sts-0         1/1     Running   0                16m    10.244.1.91    worker2   <none>           <none>
altibase-sts-1         1/1     Running   0                16m    10.244.2.118   worker1   <none>           <none>
altibase-sts-2         1/1     Running   0                15m    10.244.1.92    worker2   <none>           <none>
altibase-sts-3         1/1     Running   0                14m    10.244.2.119   worker1   <none>           <none>
```

##### Check Altibase replication operation
- The altibase-sts-3 pod connects to the local Altibase with isql and inputs data to the T1 table.
- Access the altibase of the altibase-sts-2 pod with isql from the altibase-sts-3 pod and search the T1 table.

```
$ kubectl exec -it altibase-sts-3 -- /bin/bash
root@altibase-sts-3:/# . /home/altibase/config_map/set_altibase.env
root@altibase-sts-3:/# is
-----------------------------------------------------------------
     Altibase Client Query utility.
     Release Version 7.1.0.8.8
     Copyright 2000, ALTIBASE Corporation or its subsidiaries.
     All Rights Reserved.
-----------------------------------------------------------------
ISQL_CONNECTION = TCP, SERVER = localhost, PORT_NO = 20300
iSQL> insert into t1 values(1,1);
iSQL> exit

root@altibase-sts-3:/# isql -s altibase-sts-2.altibase-svc -u sys -p manager
-----------------------------------------------------------------
     Altibase Client Query utility.
     Release Version 7.1.0.8.8
     Copyright 2000, ALTIBASE Corporation or its subsidiaries.
     All Rights Reserved.
-----------------------------------------------------------------
ISQL_CONNECTION = TCP, SERVER = altibase-sts-2.altibase-svc, PORT_NO = 20300
iSQL> select * from T1;
I1          I2
---------------------------
1           1
1 row selected.
```

##### Check Scale-down operation

- Scale-down with replicas=3.
- Connect to the altibase-sts-2 pod.
- Check that the XSN value of replication AKU_REP_23 is -1, which is the reset state.

```
$ kubectl scale sts altibase-sts --replicas=3
statefulset.apps/altibase-sts scaled
$
$ kubectl exec -it altibase-sts-2 -- /bin/bash
root@altibase-sts-2:/# . /home/altibase/config_map/set_altibase.env
root@altibase-sts-2:/# is
-----------------------------------------------------------------
     Altibase Client Query utility.
     Release Version 7.1.0.8.8
     Copyright 2000, ALTIBASE Corporation or its subsidiaries.
     All Rights Reserved.
-----------------------------------------------------------------
ISQL_CONNECTION = TCP, SERVER = localhost, PORT_NO = 20300
iSQL> select REPLICATION_NAME, XSN from system_.sys_replications_;
REPLICATION_NAME                          XSN
------------------------------------------------------------------
AKU_REP_02                                1591138
AKU_REP_12                                1591138
AKU_REP_23                                -1
3 rows selected.
```

##### Check Scale-up operation

- Connect to the altibase-sts-0 pod and insert additional data into the T1 table.
- Scale-up with replicas=4.
- Connect to the altibase-sts-3 pod and check if the data additionally inserted in the T1 table is reflected in the scaled-up pod.

```
$ kubectl exec -it altibase-sts-0 -- /bin/bash
root@altibase-sts-0:/# . /home/altibase/config_map/set_altibase.env
root@altibase-sts-0:/# is
-----------------------------------------------------------------
     Altibase Client Query utility.
     Release Version 7.1.0.8.8
     Copyright 2000, ALTIBASE Corporation or its subsidiaries.
     All Rights Reserved.
-----------------------------------------------------------------
ISQL_CONNECTION = TCP, SERVER = localhost, PORT_NO = 20300
iSQL> insert into t1 values(2,2);
1 row inserted.
iSQL> exit
root@altibase-sts-0:/# exit
$
$ kubectl scale sts altibase-sts --replicas=4
statefulset.apps/altibase-sts scaled
$
$ kubectl exec -it altibase-sts-3 -- /bin/bash
root@altibase-sts-3:/# . /home/altibase/config_map/set_altibase.env
root@altibase-sts-3:/# is
-----------------------------------------------------------------
     Altibase Client Query utility.
     Release Version 7.1.0.8.8
     Copyright 2000, ALTIBASE Corporation or its subsidiaries.
     All Rights Reserved.
-----------------------------------------------------------------
ISQL_CONNECTION = TCP, SERVER = localhost, PORT_NO = 20300
iSQL> select * from T1;
I1          I2
---------------------------
1           1
2           2
2 row selected.
```
