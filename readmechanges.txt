kubectl apply -f /home/sam/vscode/coffeacasa-singleuser/kuantifierJupyter/exporter.yaml -n cmsaf-prod
#helm upgrade --install kuantifier oci://hub.opensciencegrid.org/iris-hep/kuantifier --version 0.3.0 --namespace cmsaf-prod --values /home/sam/vscode/coffeacasa-singleuser/kuantifierJupyter/helm-values.yaml


helm upgrade --install kuantifier /home/sam/vscode/kapel/chart \
  -n monitoring \
  -f /home/sam/vscode/kapel/chart/values.yaml \
  --atomic --debug

#######################################################################

KAPEL for JupyterLab
Custom Metrics Exporter Integration

This updated kapel makes use of custom metrics to include all jupyterhub pods that are missing kube_pod_completion times and kube_pod_container_resource_requests.

kapel_pod_last_seen - placeholder for kube_pod_completion_time becomes kapel_pod_endtime if the pod stops
kapel_pod_endtime - End time when pod disappears or completes, 
kapel_pod_cpu_requests - cpu requests grabbed from pod specs

exporter.yaml

The exporter adds supplemental Prometheus metrics to ensure every pod is tracked:

custom metric kapel_pod_cpu_requests which is just cpu_cores = float(pod.spec.containers[0].resources.requests["cpu"])
this was set to ensure that we have something calculate cpu resources with if it's missing on jupyter pod

kapel_pod_last_seen times Timestamp when the pod was last observed.

kapel_pod_endtime end time written when a pod disappears or stops reporting.

kapel_pod_cpu_requests cpu requests pulled directly from the pod spec.

query changes

#prefer the kube_pod_completion to fall back to kapel_pod_endtime if it's missing

(max_over_time(kube_pod_completion_time{namespace="$NS"}[$RANGE])
 or on(pod, uid, namespace)
 max_over_time(kapel_pod_endtime{namespace="$NS"}[$RANGE]))


#prefer the kube_pod_container_resource_requests to fall back to kapel_pod_cpu_requests if it's missing
(max_over_time(kube_pod_container_resource_requests{resource="cpu", node!="" ,namespace="$NS"}[$RANGE])
 or on(pod, uid, namespace)
 max_over_time(kapel_pod_cpu_requests{namespace="$NS"}[$RANGE]))


same query adjusted to get the duration the pod ran end - start with fallbacks if the KSM scraped metrics are missing

(max by (pod, uid)(
  (max_over_time(kube_pod_completion_time{namespace="$NS"}[$RANGE])
   or on(pod, uid, namespace)
   max_over_time(kapel_pod_endtime{namespace="$NS"}[$RANGE]))
) -
max by (pod, uid)(
  max_over_time(kube_pod_start_time{namespace="$NS"}[$RANGE])
))
* on(pod,uid) group_left()
max by (pod,uid)(
  (max_over_time(kube_pod_container_resource_requests{resource="cpu",node!="",namespace="$NS"}[$RANGE])
   or on(pod,uid,namespace)
   max_over_time(kapel_pod_cpu_requests{namespace="$NS"}[$RANGE]))
)
