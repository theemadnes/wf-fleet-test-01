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

export VPC_1=shared-vpc-01
export VPC_2=shared-vpc-02

export REGION_1=us-central1
export REGION_2=us-east4

export GKE_SVC_1_01=gke-svc01-${REGION_1}
export GKE_SVC_1_02=gke-svc01-${REGION_2}
export GKE_SVC_2_01=gke-svc02-${REGION_1}
export GKE_SVC_2_02=gke-svc02-${REGION_2}

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

# manual VPC creation
gcloud compute networks create $VPC_1 --project=wf-host-01 --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional
gcloud compute networks subnets create sub-01-us-central1 --project=wf-host-01 --range=10.128.0.0/20 --stack-type=IPV4_ONLY --network=$VPC_1 --region=us-central1 --enable-private-ip-google-access
gcloud compute networks subnets create sub-01-us-east4 --project=wf-host-01 --range=10.150.0.0/20 --stack-type=IPV4_ONLY --network=$VPC_1 --region=us-east4 --enable-private-ip-google-access

gcloud compute networks create $VPC_2 --project=wf-host-01 --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional
gcloud compute networks subnets create sub-02-us-central1 --project=wf-host-01 --range=10.128.0.0/20 --stack-type=IPV4_ONLY --network=$VPC_2 --region=us-central1 --enable-private-ip-google-access
gcloud compute networks subnets create sub-02-us-east4 --project=wf-host-01 --range=10.150.0.0/20 --stack-type=IPV4_ONLY --network=$VPC_2 --region=us-east4 --enable-private-ip-google-access

# enable shared VPC (after granting Shared VPC Admin role)
gcloud beta compute shared-vpc enable $HOST_PROJECT

# attach service projects
gcloud beta compute shared-vpc associated-projects add $SVC_PROJECT_1 \
--host-project $HOST_PROJECT

gcloud beta compute shared-vpc associated-projects add $SVC_PROJECT_2 \
--host-project $HOST_PROJECT
```