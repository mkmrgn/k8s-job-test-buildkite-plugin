# k8s-job-buildkite-plugin

Run a [Command Step](https://buildkite.com/docs/pipelines/command-step) as a Kubernetes [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) using a [Pod Spec](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec).

## RBAC

You will need to ensure your agent deployment has sufficient privileges to create resources on your kubernetes cluster. An example RBAC policy:

```yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  name: agent-k8s-job
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: agent-k8s-job
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "watch", "list"]
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: agent-k8s-job
subjects:
  - kind: ServiceAccount
    name: agent-k8s-job
roleRef:
  kind: Role
  name: agent-k8s-job
  apiGroup: rbac.authorization.k8s.io
```

## Configuration

`pod-spec` this takes a [kubernetes pod spec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#podspec-v1-core) that will run in a kubernetes job. It is often easier to format this with a yaml reference like below:

```yaml
success-spec: &success-spec
  restartPolicy: OnFailure
  containers:
    - name: ${BUILDKITE_PIPELINE_SLUG}-${BUILDKITE_BUILD_NUMBER}-${BUILDKITE_STEP_ID}-success
      image: deftinc/winpenguin
      command: ["/app/entrypoint.sh"]

steps:
  - label: "Consulting my :crystal_ball: thats inside my :k8s: pod"
    plugins:
      - k8s-job:
          pod-spec: *success-spec
```

`cleanup` will clean up the job and pod resources after the step runs. If set to false they will remain for 10 minutes to debug. Defaults to true.

`metadata.annotations` will add metadata (annotations) to the kubernetes job. Defaults to an empty object.

`mount-source` will mount the source code to `/buildkite/src` if set to true. This can be used for build steps that build images from source. Defaults to false.

To use this option your agent must include in the nodeName it is running on in its environment variables as `BUILDKITE_AGENT_NODE_NAME`. It must also include a hostPath volumes and volumeMount for the agent.

```
env:
  - name: AGENT_NODE_NAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName
```

```
volumes:
  - name: builds
    hostPath:
      path: /data/buildkite/builds
      type: DirectoryOrCreate
```

```
volumeMounts:
  - name: builds
    mountPath: "/buildkite/builds"
```
