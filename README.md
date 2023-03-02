# Cert Manager Blueprints Deployment Guide

## Introduction

Cert Manager adds certificates and certificate issuers as resource types in Kubernetes clusters, and simplifies the process of obtaining, renewing and using those certificates.

## Why?
As users of official cert manager should deploy their resources like 'Certificate, ClusterIssuer' manually, considering that, this helm chart will help to deploy cert manager with some features such as Certificate and ClusterIssuer on K8S cluster.

## How?
When deploying this chart, first, the crds and cert manager of official cert manager helm chart are being deployed, after that, in the same namespace where you deployed this chart, the resources ClusterIssuer, Certificate and Secret will be deployed.

## Cert Manager version updates
Workflow will check every sunday at 12am if there is newer version of cert-manager dependency. If no, nothing will be changed, if yes the dependency will be updated after which pushed to the repository.

## Templates descriptions
> __Note__
The keywords are not explained below, as you can see them simply typing **kubectl explain resource kind**.

##### clusterissuer-acme.yaml
It is preferred to deploy ClusterIssuer instead of Issuer, that it be more globally for cluster. apiVersion is CRD api, you should have deployed cert-manager with its CRDs on your cluster, or just enable dependency in this chart. As this chart is only for AWS Route53, the clusterIssuer solver is route53. **In future, maybe added other solvers based on user requirements.** Let's say (imagine) that by this template you will bring Certificate Authority to your cluster, after which the **certificate.yaml** file will use this template's CA's configurations to create a certificate for you.
<br>
<p>
  
```yaml
{{- with .Values }}
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: "{{ $.Release.Name }}-route53-clusterissuer"
  namespace: {{ .namespace | quote }}
spec:
  acme:
    {{- if .email }}
    email: {{ .email | quote }}
    {{- end }}
    {{- if .preferredChain }}
    preferredChain: {{ .preferredChain | quote }}
    {{- end }}
    {{- if .externalAccountBinding.keyID }}
    externalAccountBinding:
      keyID: {{ .externalAccountBinding.keyID | quote }}
      keySecretRef:
        key: secret
        name: {{ $.Release.Name }}-acme-server-secretkey
    {{- end }}
    privateKeySecretRef:
      name: "{{ $.Release.Name }}-route53-secretkeyref"
    server: {{ .acmeServerUrl | quote }}
    solvers:
    - dns01:
        route53:
          {{- if .hostedZoneID }}
          hostedZoneID: {{ .hostedZoneID | quote }}
          {{- end }}
          region: {{ .region | default "global" | quote }}
      {{- if .dnsZones }}
      selector:
        dnsZones:
        {{- .dnsZones | toYaml | nindent 12 }}
      {{- end }}
{{- end }}
```
  
</p>
<br>

##### certificate.yaml
This is the certificate template, after deployment of this file it will issue your deployed **clusterIssuer**. The values of this template are simple, it requires only the CName to include in your certificate & **isCA** specifying which, you will mention your certificate as valid to be signed.
<br>
<p>
  
```yaml
{{- with .Values }}
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
    name: "{{ $.Release.Name }}-certificate"
    namespace: {{ .namespace | quote }}
spec:
  commonName: {{ .commonName | quote }}
  isCA: {{ .isCA }}
  issuerRef:
    kind: "ClusterIssuer"
    name: "{{ $.Release.Name }}-route53-clusterissuer"
  secretName: "{{ $.Release.Name }}-certificate-secret"
  {{ if .dnsZones }}
  dnsNames:
  {{ .dnsZones | toYaml | indent 4 }}
  {{ end }}
{{- end -}}
```
  
</p>
<br>

##### acme-server-secretkey-secret.yaml
This template will be generated only in case, if your Certificate Authority does not give a certificate for you without specifying CA's token. If you are using CA such as ZeroSSL, it will give you a token to specify while issueing a certificate to authorize that only you are issueing a certificate from your own ZeroSSL account.
<br>
<p>
  
```yaml
{{- if .Values.externalAccountBinding.secretKey }}
{{- with .Values }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ $.Release.Name }}-acme-server-secretkey
  namespace: {{ .namespace | quote }}
stringData:
  secret: {{ .externalAccountBinding.secretKey | quote }}
{{- end }}
{{- end }}
```
  
</p>
<br>
