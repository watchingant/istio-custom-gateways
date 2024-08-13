# How to Setup External CA in Istio
This repository contains steps and source code for configuring an external CA in Istio. 

## Prerequisites
* [AWS Account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/) (You only need an AWS account if you intend to use AWS Private CA)
* [AWS Private CA](https://aws.amazon.com/private-ca/)
* Kubernetes cluster
* [kubectl](https://kubernetes.io/docs/tasks/tools/)
* [istioctl](https://istio.io/latest/docs/setup/getting-started/#download)

## Create Private CA in AWS Private CA
If you have an AWS account, head over to AWS Private CA to create a Certificate Authority that you'll use. 

## Download Public Root Certificate from AWS Private CA
```
aws acm-pca get-certificate-authority-certificate --certificate-authority-arn <certificate-authority-arn> --region af-south-1 --output text > ca.pem
```

## Create Secret for Root Certificate
```
kubectl create secret generic -n cert-manager istio-root-ca --from-file=ca.pem=ca.pem
```

## Install Cert Manager (kubectl or Helm)
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.2/cert-manager.yaml
```
OR
```
helm repo add jetstack https://charts.jetstack.io --force-update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.15.2 \
  --set crds.enabled=true
```

## Install AWS Private CA Issuer Plugin
```
kubectl create namespace aws-pca-issuer

helm repo add awspca https://cert-manager.github.io/aws-privateca-issuer
helm repo update
helm install awspca/aws-privateca-issuer  --generate-name --namespace aws-pca-issuer
```

## Set EKS Node Permissions for AWS Private CA
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "awspcaissuer",
            "Action": [
                "acm-pca:DescribeCertificateAuthority",
                "acm-pca:GetCertificate",
                "acm-pca:IssueCertificate"
            ],
			"Effect": "Allow",
            "Resource": "arn:aws:acm-pca:<region>:<account_id>:certificate-authority/<resource_id>"
        }       
    ]
}
```

## Create an Issuer (AWSPCAClusterIssuer) in Kubernetes Cluster
The issuer represents the CA and is used to sign istiod and mesh workload certificates. It will communicate with AWS Private CA.
```
apiVersion: awspca.cert-manager.io/v1beta1
kind: AWSPCAClusterIssuer
metadata:
  name: aws-pca-root-ca
spec:
  arn: arn:aws:acm-pca:af-south-1:<aws-account-id>:certificate-authority/9144c309...
  region: af-south-1
```

## Create `istio-system` Namespace
This is where the `istiod` certificate will live.
```
kubectl create ns istio-system
```

## Install Istio CSR Configured with AWS Private CA Issuer Plugin
Some of the custom helm values used in this command are only relevant for AWS Private CA issuer. You can update it accordingly depending on your external CA.
```
helm install -n cert-manager cert-manager-istio-csr jetstack/cert-manager-istio-csr \
	--set "app.certmanager.issuer.group=awspca.cert-manager.io" \
	--set "app.certmanager.issuer.kind=AWSPCAClusterIssuer" \
	--set "app.certmanager.issuer.name=aws-pca-root-ca" \
	--set "app.certmanager.preserveCertificateRequests=true" \
	--set "app.server.maxCertificateDuration=48h" \
	--set "app.tls.certificateDuration=24h" \
	--set "app.tls.istiodCertificateDuration=24h" \
	--set "app.tls.rootCAFile=/var/run/secrets/istio-csr/ca.pem" \
	--set "volumeMounts[0].name=root-ca" \
	--set "volumeMounts[0].mountPath=/var/run/secrets/istio-csr" \
	--set "volumes[0].name=root-ca" \
	--set "volumes[0].secret.secretName=istio-root-ca"
```

## Install Istio with Custom Configurations
```
istioctl operator init
kubectl apply -f istio-custom-config.yaml
```
