# wf-fleet-test-01
some mesh, shared VPC and project testing

### project setup

```
export HOST_PROJECT=wf-host-01
export SVC_PROJECT_1=wf-service-01
export SVC_PROJECT_2=wf-service-02

export HOST_PROJECT_NUM=$(gcloud projects describe ${HOST_PROJECT} --format="value(projectNumber)")
export SVC_PROJECT_1_NUM=$(gcloud projects describe ${SVC_PROJECT_1} --format="value(projectNumber)")
export SVC_PROJECT_2_NUM=$(gcloud projects describe ${SVC_PROJECT_2} --format="value(projectNumber)")

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

# set up fleet permissions
GKE_PROJECT_ID=$SVC_PROJECT_1
FLEET_HOST_PROJECT_ID=$HOST_PROJECT
FLEET_HOST_PROJECT_NUMBER=$(gcloud projects describe "${FLEET_HOST_PROJECT_ID}" --format "value(projectNumber)")
gcloud projects add-iam-policy-binding "${FLEET_HOST_PROJECT_ID}" \
  --member "serviceAccount:service-${FLEET_HOST_PROJECT_NUMBER}@gcp-sa-gkehub.iam.gserviceaccount.com" \
  --role roles/gkehub.serviceAgent
gcloud projects add-iam-policy-binding "${GKE_PROJECT_ID}" \
  --member "serviceAccount:service-${FLEET_HOST_PROJECT_NUMBER}@gcp-sa-gkehub.iam.gserviceaccount.com" \
  --role roles/gkehub.serviceAgent
gcloud projects add-iam-policy-binding "${GKE_PROJECT_ID}" \
  --member "serviceAccount:service-${FLEET_HOST_PROJECT_NUMBER}@gcp-sa-gkehub.iam.gserviceaccount.com" \
  --role roles/gkehub.crossProjectServiceAgent

GKE_PROJECT_ID=$SVC_PROJECT_2
FLEET_HOST_PROJECT_NUMBER=$(gcloud projects describe "${FLEET_HOST_PROJECT_ID}" --format "value(projectNumber)")
gcloud projects add-iam-policy-binding "${FLEET_HOST_PROJECT_ID}" \
  --member "serviceAccount:service-${FLEET_HOST_PROJECT_NUMBER}@gcp-sa-gkehub.iam.gserviceaccount.com" \
  --role roles/gkehub.serviceAgent
gcloud projects add-iam-policy-binding "${GKE_PROJECT_ID}" \
  --member "serviceAccount:service-${FLEET_HOST_PROJECT_NUMBER}@gcp-sa-gkehub.iam.gserviceaccount.com" \
  --role roles/gkehub.serviceAgent
gcloud projects add-iam-policy-binding "${GKE_PROJECT_ID}" \
  --member "serviceAccount:service-${FLEET_HOST_PROJECT_NUMBER}@gcp-sa-gkehub.iam.gserviceaccount.com" \
  --role roles/gkehub.crossProjectServiceAgent

# manual VPC creation
gcloud compute networks create $VPC_1 --project=wf-host-01 --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional
gcloud compute networks subnets create sub-01-us-central1 --project=wf-host-01 --range=10.128.0.0/20 --stack-type=IPV4_ONLY --network=$VPC_1 --region=us-central1 --enable-private-ip-google-access
gcloud compute networks subnets create sub-01-us-east4 --project=wf-host-01 --range=10.150.0.0/20 --stack-type=IPV4_ONLY --network=$VPC_1 --region=us-east4 --enable-private-ip-google-access

gcloud compute networks create $VPC_2 --project=wf-host-01 --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional
gcloud compute networks subnets create sub-02-us-central1 --project=wf-host-01 --range=10.128.0.0/20 --stack-type=IPV4_ONLY --network=$VPC_2 --region=us-central1 --enable-private-ip-google-access
gcloud compute networks subnets create sub-02-us-east4 --project=wf-host-01 --range=10.150.0.0/20 --stack-type=IPV4_ONLY --network=$VPC_2 --region=us-east4 --enable-private-ip-google-access

# add secondary pod ranges
gcloud compute networks subnets update sub-01-us-central1 \
    --region ${REGION_1} \
    --add-secondary-ranges pod-range-us-central1="10.124.0.0/17"

gcloud compute networks subnets update sub-01-us-east4 \
    --region ${REGION_2} \
    --add-secondary-ranges pod-range-us-east4="10.54.0.0/17"

gcloud compute networks subnets update sub-02-us-central1 \
    --region ${REGION_1} \
    --add-secondary-ranges pod-range-us-central1="10.124.0.0/17"

gcloud compute networks subnets update sub-02-us-east4 \
    --region ${REGION_2} \
    --add-secondary-ranges pod-range-us-east4="10.54.0.0/17"

# add secondary service ranges 
gcloud compute networks subnets update sub-01-us-central1 \
    --region ${REGION_1} \
    --add-secondary-ranges svc-range-us-central1="192.168.124.0/24"

gcloud compute networks subnets update sub-01-us-east4 \
    --region ${REGION_2} \
    --add-secondary-ranges svc-range-us-east4="192.168.54.0/24"

gcloud compute networks subnets update sub-02-us-central1 \
    --region ${REGION_1} \
    --add-secondary-ranges svc-range-us-central1="192.168.124.0/24"

gcloud compute networks subnets update sub-02-us-east4 \
    --region ${REGION_2} \
    --add-secondary-ranges svc-range-us-east4="192.168.54.0/24"

# create super-permissive FW rule for VPC
gcloud compute --project=$HOST_PROJECT firewall-rules create all-10 --direction=INGRESS --priority=1000 --network=$VPC_1 --action=ALLOW --rules=all --source-ranges=10.0.0.0/8
gcloud compute --project=$HOST_PROJECT firewall-rules create all-10-2 --direction=INGRESS --priority=1000 --network=$VPC_2 --action=ALLOW --rules=all --source-ranges=10.0.0.0/8
gcloud compute --project=$HOST_PROJECT firewall-rules create all-192 --direction=INGRESS --priority=1000 --network=$VPC_1 --action=ALLOW --rules=all --source-ranges=10.0.0.0/8
gcloud compute --project=$HOST_PROJECT firewall-rules create all-192-2 --direction=INGRESS --priority=1000 --network=$VPC_2 --action=ALLOW --rules=all --source-ranges=10.0.0.0/8

# enable shared VPC (after granting Shared VPC Admin role)
gcloud beta compute shared-vpc enable $HOST_PROJECT

# attach service projects
gcloud beta compute shared-vpc associated-projects add $SVC_PROJECT_1 \
--host-project $HOST_PROJECT

gcloud beta compute shared-vpc associated-projects add $SVC_PROJECT_2 \
--host-project $HOST_PROJECT

# grant permissions for subnets & firewall
gcloud projects add-iam-policy-binding $HOST_PROJECT \
--member "user:admin@alexmattson.altostrat.com" \
--role "roles/compute.networkUser"

gcloud projects add-iam-policy-binding $HOST_PROJECT \
--member "serviceAccount:${SVC_PROJECT_1_NUM}@cloudservices.gserviceaccount.com" \
--role "roles/compute.networkUser"

gcloud projects add-iam-policy-binding $HOST_PROJECT \
--member "serviceAccount:service-${SVC_PROJECT_1_NUM}@container-engine-robot.iam.gserviceaccount.com" \
--role "roles/compute.networkUser"

gcloud projects add-iam-policy-binding $HOST_PROJECT \
--member "serviceAccount:${SVC_PROJECT_2_NUM}@cloudservices.gserviceaccount.com" \
--role "roles/compute.networkUser"

gcloud projects add-iam-policy-binding $HOST_PROJECT \
--member "serviceAccount:service-${SVC_PROJECT_2_NUM}@container-engine-robot.iam.gserviceaccount.com" \
--role "roles/compute.networkUser"

gcloud projects add-iam-policy-binding $HOST_PROJECT \
    --member=serviceAccount:service-${SVC_PROJECT_1_NUM}@container-engine-robot.iam.gserviceaccount.com \
    --role=roles/compute.securityAdmin

gcloud projects add-iam-policy-binding $HOST_PROJECT \
    --member=serviceAccount:service-${SVC_PROJECT_2_NUM}@container-engine-robot.iam.gserviceaccount.com \
    --role=roles/compute.securityAdmin

gcloud projects add-iam-policy-binding $HOST_PROJECT \
    --member serviceAccount:service-${SVC_PROJECT_1_NUM}@container-engine-robot.iam.gserviceaccount.com \
    --role roles/container.hostServiceAgentUser

gcloud projects add-iam-policy-binding $HOST_PROJECT \
    --member serviceAccount:service-${SVC_PROJECT_2_NUM}@container-engine-robot.iam.gserviceaccount.com \
    --role roles/container.hostServiceAgentUser

# test visibility
gcloud container subnets list-usable \
    --project $SVC_PROJECT_1 \
    --network-project $HOST_PROJECT

# create GKE clusters
gcloud --project=$SVC_PROJECT_1 container clusters create-auto \
${GKE_SVC_1_01} --region ${REGION_1} \
--release-channel rapid --labels mesh_id=proj-${HOST_PROJECT_NUM} \
--fleet-project=$HOST_PROJECT --network=projects/$HOST_PROJECT/global/networks/${VPC_1} --subnetwork=projects/$HOST_PROJECT/regions/${REGION_1}/subnetworks/sub-01-us-central1 \
--cluster-secondary-range-name=pod-range-us-central1 --services-secondary-range-name=svc-range-us-central1

```