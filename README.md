# wf-fleet-test-01
some mesh, shared VPC and project testing

### project setup

```
export HOST_PROJECT=wf-host-01
export SVC_PROJECT_1=wf-service-01
export SVC_PROJECT_2=wf-service-02

export HOST_PROJECT_NUM=$(gcloud projects describe ${HOST_PROJECT} --format="value(projectNumber)")
export SVC_PROJECT_1_NUM=$(gcloud projects describe ${SVC_PROJECT_1} --format="value(projectNumber)")
export SVC_PROJECT_2_NUM=$(gcloud projects describe ${SVC_PROJECT_1} --format="value(projectNumber)")

# initial API enablement
for PROJECT in $HOST_PROJECT $SVC_PROJECT_1 $SVC_PROJECT_2
do 
    gcloud services enable --project $PROJECT \
        compute.googleapis.com \
        container.googleapis.com \
        mesh.googleapis.com \
        gkehub.googleapis.com \
        multiclusterservicediscovery.googleapis.com \
        multiclusteringress.googleapis.com \
        trafficdirector.googleapis.com \
        certificatemanager.googleapis.com
done
```