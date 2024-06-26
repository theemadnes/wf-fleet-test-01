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

gcloud --project=$SVC_PROJECT_1 container clusters create-auto \
${GKE_SVC_1_02} --region ${REGION_2} \
--release-channel rapid --labels mesh_id=proj-${HOST_PROJECT_NUM} \
--fleet-project=$HOST_PROJECT --network=projects/$HOST_PROJECT/global/networks/${VPC_1} --subnetwork=projects/$HOST_PROJECT/regions/${REGION_2}/subnetworks/sub-01-us-east4 \
--cluster-secondary-range-name=pod-range-us-east4 --services-secondary-range-name=svc-range-us-east4

gcloud --project=$SVC_PROJECT_2 container clusters create-auto --async \
${GKE_SVC_2_01} --region ${REGION_1} \
--release-channel rapid --labels mesh_id=proj-${HOST_PROJECT_NUM} \
--fleet-project=$HOST_PROJECT --network=projects/$HOST_PROJECT/global/networks/${VPC_2} --subnetwork=projects/$HOST_PROJECT/regions/${REGION_1}/subnetworks/sub-02-us-central1 \
--cluster-secondary-range-name=pod-range-us-central1 --services-secondary-range-name=svc-range-us-central1

gcloud --project=$SVC_PROJECT_2 container clusters create-auto --async \
${GKE_SVC_2_02} --region ${REGION_2} \
--release-channel rapid --labels mesh_id=proj-${HOST_PROJECT_NUM} \
--fleet-project=$HOST_PROJECT --network=projects/$HOST_PROJECT/global/networks/${VPC_2} --subnetwork=projects/$HOST_PROJECT/regions/${REGION_2}/subnetworks/sub-02-us-east4 \
--cluster-secondary-range-name=pod-range-us-east4 --services-secondary-range-name=svc-range-us-east4

# forgot, so enable GKE Enterprise
gcloud services enable --project=$HOST_PROJECT \
   anthos.googleapis.com \
   gkehub.googleapis.com

# enable CSM on all clusters in fleet
gcloud container fleet mesh enable

gcloud container fleet mesh update \
    --management automatic \
    --memberships ${GKE_SVC_1_01},${GKE_SVC_1_02},${GKE_SVC_2_01},${GKE_SVC_2_02}
```

### result

```
$ gcloud container fleet mesh describe
createTime: '2024-03-27T07:34:28.763072459Z'
membershipSpecs:
  projects/321878386322/locations/us-central1/memberships/gke-svc01-us-central1:
    mesh:
      management: MANAGEMENT_AUTOMATIC
  projects/321878386322/locations/us-central1/memberships/gke-svc02-us-central1:
    mesh:
      management: MANAGEMENT_AUTOMATIC
  projects/321878386322/locations/us-east4/memberships/gke-svc01-us-east4:
    mesh:
      management: MANAGEMENT_AUTOMATIC
  projects/321878386322/locations/us-east4/memberships/gke-svc02-us-east4:
    mesh:
      management: MANAGEMENT_AUTOMATIC
membershipStates:
  projects/321878386322/locations/us-central1/memberships/gke-svc01-us-central1:
    servicemesh:
      controlPlaneManagement:
        details:
        - code: REVISION_FAILED_PRECONDITION
          details: An internal error has occurred. Please contact customer support.
            This will be retried within 15 minutes.
        state: FAILED_PRECONDITION
      dataPlaneManagement:
        details:
        - code: DISABLED
          details: Data Plane Management is not enabled.
        state: DISABLED
    state:
      code: ERROR
      description: 'Revision reporting unhealthy: asm-managed-rapid. Please visit
        https://cloud.google.com/service-mesh/docs/troubleshooting/troubleshoot-intro
        for details.'
      updateTime: '2024-03-27T07:42:31.379160795Z'
  projects/321878386322/locations/us-central1/memberships/gke-svc02-us-central1:
    servicemesh:
      controlPlaneManagement:
        details:
        - code: REVISION_FAILED_PRECONDITION
          details: An internal error has occurred. Please contact customer support.
            This will be retried within 15 minutes.
        state: FAILED_PRECONDITION
      dataPlaneManagement:
        details:
        - code: DISABLED
          details: Data Plane Management is not enabled.
        state: DISABLED
    state:
      code: ERROR
      description: 'Revision reporting unhealthy: asm-managed-rapid. Please visit
        https://cloud.google.com/service-mesh/docs/troubleshooting/troubleshoot-intro
        for details.'
      updateTime: '2024-03-27T07:42:42.678505172Z'
  projects/321878386322/locations/us-east4/memberships/gke-svc01-us-east4:
    servicemesh:
      controlPlaneManagement:
        details:
        - code: REVISION_FAILED_PRECONDITION
          details: An internal error has occurred. Please contact customer support.
            This will be retried within 15 minutes.
        state: FAILED_PRECONDITION
      dataPlaneManagement:
        details:
        - code: DISABLED
          details: Data Plane Management is not enabled.
        state: DISABLED
    state:
      code: ERROR
      description: 'Revision reporting unhealthy: asm-managed-rapid. Please visit
        https://cloud.google.com/service-mesh/docs/troubleshooting/troubleshoot-intro
        for details.'
      updateTime: '2024-03-27T07:42:38.254957403Z'
  projects/321878386322/locations/us-east4/memberships/gke-svc02-us-east4:
    servicemesh:
      controlPlaneManagement:
        details:
        - code: REVISION_FAILED_PRECONDITION
          details: An internal error has occurred. Please contact customer support.
            This will be retried within 15 minutes.
        state: FAILED_PRECONDITION
      dataPlaneManagement:
        details:
        - code: DISABLED
          details: Data Plane Management is not enabled.
        state: DISABLED
    state:
      code: ERROR
      description: 'Revision reporting unhealthy: asm-managed-rapid. Please visit
        https://cloud.google.com/service-mesh/docs/troubleshooting/troubleshoot-intro
        for details.'
      updateTime: '2024-03-27T07:42:33.281805608Z'
name: projects/wf-host-01/locations/global/features/servicemesh
resourceState:
  state: ACTIVE
spec: {}
updateTime: '2024-03-27T07:42:49.606948856Z'
admin_@cloudshell:~/wf-fleet-test-01 (wf-host-01)$ gcloud container fleet mesh describe
createTime: '2024-03-27T07:34:28.763072459Z'
membershipSpecs:
  projects/321878386322/locations/us-central1/memberships/gke-svc01-us-central1:
    mesh:
      management: MANAGEMENT_AUTOMATIC
  projects/321878386322/locations/us-central1/memberships/gke-svc02-us-central1:
    mesh:
      management: MANAGEMENT_AUTOMATIC
  projects/321878386322/locations/us-east4/memberships/gke-svc01-us-east4:
    mesh:
      management: MANAGEMENT_AUTOMATIC
  projects/321878386322/locations/us-east4/memberships/gke-svc02-us-east4:
    mesh:
      management: MANAGEMENT_AUTOMATIC
membershipStates:
  projects/321878386322/locations/us-central1/memberships/gke-svc01-us-central1:
    servicemesh:
      controlPlaneManagement:
        details:
        - code: REVISION_FAILED_PRECONDITION
          details: An internal error has occurred. Please contact customer support.
            This will be retried within 15 minutes.
        state: FAILED_PRECONDITION
      dataPlaneManagement:
        details:
        - code: DISABLED
          details: Data Plane Management is not enabled.
        state: DISABLED
    state:
      code: ERROR
      description: 'Revision reporting unhealthy: asm-managed-rapid. Please visit
        https://cloud.google.com/service-mesh/docs/troubleshooting/troubleshoot-intro
        for details.'
      updateTime: '2024-03-27T07:42:31.379160795Z'
  projects/321878386322/locations/us-central1/memberships/gke-svc02-us-central1:
    servicemesh:
      controlPlaneManagement:
        details:
        - code: REVISION_FAILED_PRECONDITION
          details: An internal error has occurred. Please contact customer support.
            This will be retried within 15 minutes.
        state: FAILED_PRECONDITION
      dataPlaneManagement:
        details:
        - code: DISABLED
          details: Data Plane Management is not enabled.
        state: DISABLED
    state:
      code: ERROR
      description: 'Revision reporting unhealthy: asm-managed-rapid. Please visit
        https://cloud.google.com/service-mesh/docs/troubleshooting/troubleshoot-intro
        for details.'
      updateTime: '2024-03-27T07:42:42.678505172Z'
  projects/321878386322/locations/us-east4/memberships/gke-svc01-us-east4:
    servicemesh:
      controlPlaneManagement:
        details:
        - code: REVISION_FAILED_PRECONDITION
          details: An internal error has occurred. Please contact customer support.
            This will be retried within 15 minutes.
        state: FAILED_PRECONDITION
      dataPlaneManagement:
        details:
        - code: DISABLED
          details: Data Plane Management is not enabled.
        state: DISABLED
    state:
      code: ERROR
      description: 'Revision reporting unhealthy: asm-managed-rapid. Please visit
        https://cloud.google.com/service-mesh/docs/troubleshooting/troubleshoot-intro
        for details.'
      updateTime: '2024-03-27T07:42:38.254957403Z'
  projects/321878386322/locations/us-east4/memberships/gke-svc02-us-east4:
    servicemesh:
      controlPlaneManagement:
        details:
        - code: REVISION_FAILED_PRECONDITION
          details: An internal error has occurred. Please contact customer support.
            This will be retried within 15 minutes.
        state: FAILED_PRECONDITION
      dataPlaneManagement:
        details:
        - code: DISABLED
          details: Data Plane Management is not enabled.
        state: DISABLED
    state:
      code: ERROR
      description: 'Revision reporting unhealthy: asm-managed-rapid. Please visit
        https://cloud.google.com/service-mesh/docs/troubleshooting/troubleshoot-intro
        for details.'
      updateTime: '2024-03-27T07:42:33.281805608Z'
name: projects/wf-host-01/locations/global/features/servicemesh
resourceState:
  state: ACTIVE
spec: {}
updateTime: '2024-03-27T07:42:49.606948856Z'
admin_@cloudshell:~/wf-fleet-test-01 (wf-host-01)$ gcloud container fleet mesh describe
createTime: '2024-03-27T07:34:28.763072459Z'
membershipSpecs:
  projects/321878386322/locations/us-central1/memberships/gke-svc01-us-central1:
    mesh:
      management: MANAGEMENT_AUTOMATIC
  projects/321878386322/locations/us-central1/memberships/gke-svc02-us-central1:
    mesh:
      management: MANAGEMENT_AUTOMATIC
  projects/321878386322/locations/us-east4/memberships/gke-svc01-us-east4:
    mesh:
      management: MANAGEMENT_AUTOMATIC
  projects/321878386322/locations/us-east4/memberships/gke-svc02-us-east4:
    mesh:
      management: MANAGEMENT_AUTOMATIC
membershipStates:
  projects/321878386322/locations/us-central1/memberships/gke-svc01-us-central1:
    servicemesh:
      controlPlaneManagement:
        details:
        - code: REVISION_FAILED_PRECONDITION
          details: An internal error has occurred. Please contact customer support.
            This will be retried within 15 minutes.
        state: FAILED_PRECONDITION
      dataPlaneManagement:
        details:
        - code: DISABLED
          details: Data Plane Management is not enabled.
        state: DISABLED
    state:
      code: ERROR
      description: 'Revision reporting unhealthy: asm-managed-rapid. Please visit
        https://cloud.google.com/service-mesh/docs/troubleshooting/troubleshoot-intro
        for details.'
      updateTime: '2024-03-27T07:42:31.379160795Z'
  projects/321878386322/locations/us-central1/memberships/gke-svc02-us-central1:
    servicemesh:
      controlPlaneManagement:
        details:
        - code: REVISION_FAILED_PRECONDITION
          details: An internal error has occurred. Please contact customer support.
            This will be retried within 15 minutes.
        state: FAILED_PRECONDITION
      dataPlaneManagement:
        details:
        - code: DISABLED
          details: Data Plane Management is not enabled.
        state: DISABLED
    state:
      code: ERROR
      description: 'Revision reporting unhealthy: asm-managed-rapid. Please visit
        https://cloud.google.com/service-mesh/docs/troubleshooting/troubleshoot-intro
        for details.'
      updateTime: '2024-03-27T07:42:42.678505172Z'
  projects/321878386322/locations/us-east4/memberships/gke-svc01-us-east4:
    servicemesh:
      controlPlaneManagement:
        details:
        - code: REVISION_FAILED_PRECONDITION
          details: An internal error has occurred. Please contact customer support.
            This will be retried within 15 minutes.
        state: FAILED_PRECONDITION
      dataPlaneManagement:
        details:
        - code: DISABLED
          details: Data Plane Management is not enabled.
        state: DISABLED
    state:
      code: ERROR
      description: 'Revision reporting unhealthy: asm-managed-rapid. Please visit
        https://cloud.google.com/service-mesh/docs/troubleshooting/troubleshoot-intro
        for details.'
      updateTime: '2024-03-27T07:42:38.254957403Z'
  projects/321878386322/locations/us-east4/memberships/gke-svc02-us-east4:
    servicemesh:
      controlPlaneManagement:
        details:
        - code: REVISION_FAILED_PRECONDITION
          details: An internal error has occurred. Please contact customer support.
            This will be retried within 15 minutes.
        state: FAILED_PRECONDITION
      dataPlaneManagement:
        details:
        - code: DISABLED
          details: Data Plane Management is not enabled.
        state: DISABLED
    state:
      code: ERROR
      description: 'Revision reporting unhealthy: asm-managed-rapid. Please visit
        https://cloud.google.com/service-mesh/docs/troubleshooting/troubleshoot-intro
        for details.'
      updateTime: '2024-03-27T07:42:33.281805608Z'
name: projects/wf-host-01/locations/global/features/servicemesh
resourceState:
  state: ACTIVE
spec: {}
updateTime: '2024-03-27T07:42:49.606948856Z'
```