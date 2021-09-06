## **Step:1**


`
Install VPC, IGW, NAT-GW, Subnets & Routing Tables using Terraform
`

## **Step:2**


-  Update VPC an Subnet info in the cluster config file
- > Install EKS Cluster using eksctl

        Verify Cluster:
                eksctl get cluster
        Verify nodegroup:
                eksctl get nodegroup --cluster CP-Cluster



## **Step:3**

`
Create S3 Bucket using Terraform
`

## **Step:4**

`
Install Add on's
- ALB

    - `alb.sh`

         Applies `cert-manager`
         
         Creates `ALB`
- Auto Scalar

    - Deply Cluster-autoscaler

          kubectl apply -f autoscaler-autodiscover.yaml
                        Note: Edit the file to update <YOUR CLUSTER>
    - Patch the deployment

          kubectl patch deployment cluster-autoscaler -n kube-system -p '{"spec":{"template":{"metadata":{"annotations":{"cluster-autoscaler.kubernetes.io/safe-to-evict": "false"}}}}}'
    - Edit the deployment

          `kubectl -n kube-system edit deployment.apps/cluster-autoscaler`

    -   Update with your cluster name   as      shown below:  

            --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<YOUR CLUSTER NAME>
          ---TO---

            --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/CP-Cluster
    - Set Auto Scalar cluster image

          'kubectl set image deployment cluster-autoscaler \
            -n kube-system \
             cluster-autoscaler=k8s.gcr.io/autoscaling/cluster-autoscaler:v1.20.0'
- Auto Scalar
    -   Deploy Metrics Server

            Execute `metrics-server.sh`

`
## **Step:5**

`
Install ECR using Terraform
`

-   Create ECR Repo

        Terraform apply

-   Docker Image tag/push using the given option

        aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 476262889068.dkr.ecr.us-east-1.amazonaws.com

        docker build -t cp-repo .

        docker tag cp-repo:latest 476262889068.dkr.ecr.us-east-1.amazonaws.com/cp-repo:upg-loadme

        docker push 476262889068.dkr.ecr.us-east-1.amazonaws.com/cp-repo:upg-loadme


## **Step:6**

`
Create Nodegroup using eksctl
`

-   Create Nodegroups

        eksctl create nodegroup -f cp-cluster.yaml

                List nodegroups:
                        eksctl get nodegroups --cluster CP-Cluster

## **Step:7**

`
Deploy upg-loadme
`

- Deploy POD using kubectl

        kubectl apply -f 1_name-space.yaml
        kubectl apply -f 2_service.yaml
        kubectl apply -f 3_ingress.yaml
            NOTE: update public subnetid's here before applying
        kubectl apply -f 4_deployment.yaml
            NOTE: update the image URI with your image 
- Verify:
        
        kubectl get service -n demo
        kubectl get service upg-loadme-service -n demo

        Get URL address:

                kubectl describe ing -n demo

## **Step:8**

`
Deploy Redis POD(server & CLI)
`

- Deploy Redis Server

            kubectl apply -f 1_service.yaml
            kubectl apply -f 2_configmap.yaml
            kubectl apply -f 3_statefulset.yaml
            kubectl apply -f 4_cli-redis-pod.yaml


- Deploy CLI Server

           kubectl apply -f cli-pod.yaml 

## **Step:9**

`
Install Autoscalar for upg-loadme

        sh create_hpa.sh

Increast Load:

        ab -n 100 -c 100 'http://aws-load-balancer-controller-2070989568.us-east-1.elb.amazonaws.com/root/load?scale=100'


`

## **Step:10**


`

Deploy Prometheus:

                helm install prometheus prometheus-community/prometheus --namespace demo --set alertmanager.persistentVolume.storageClass="gp2" --set server.persistentVolume.storageClass="gp2"

Check Prometheus components deployed as expected
                
                kubectl get all -n prometheus

Prometheus port forwarding

                kubectl port-forward -n prometheus deploy/prometheus-server 8080:9090

Deploy Grafana:

                kubectl -f /root/CP-Project/promotheus/grafana

Grafana port forward

                kubectl -n demo port-forward svc/grafana 3000:3000


Access dashboard IDs

                3119/6417


Check raw resource usage:

        kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"


`

## **Verify Commands**

`
Auto scalar
`
-       To view auto scalar logs:
                        kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler