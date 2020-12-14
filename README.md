# K8s and Azure KeyVault with Pod Identity Microhack


The purpose of this microhack is to access Azure KeyVault secrets/keys/certificates inside a pod leveraging Azure Pod Identity.
This microhack is based on vanilla K8s running in Azure.

## Pre-requisites

- Azure KeyVault:
    - name of the KeyVault
    - name of the Resource Group KeyVault resides in
- K8s cluster running in Azure Cloud:
    - proper /etc/default/kubelet (for DEBs), or /etc/sysconfig/kubelet (for RPMs) in place
    - proper /etc/kubernetes/azure.json in place
    - version v1.16.0+
    - RBAC enabled
- helm3
- kubectl

As an example in this microhack I have a Resource Group named `k8s-kv` a KeyVault named `k8s-kv1`.

## Useful links

- https://github.com/Azure/secrets-store-csi-driver-provider-azure
- https://github.com/Azure/secrets-store-csi-driver-provider-azure/blob/master/docs/pod-identity-mode.md

## 1. Install secrets-store-csi-driver-provider-azure

```bash
helm repo add csi-secrets-store-provider-azure https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/charts
helm repo update
helm install csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --generate-name
```

## 2. create a key/secret/certificate in your Azure KeyVault

For this microhack we'll use certificates.
Generate a certificate with the name `testcert1` in the KeyVault.

```bash
az keyvault certificate create --vault-name "k8s-kv1" -n testcert1 -p "$(az keyvault certificate get-default-policy)"
```

## 3. Create an Azure User Identity

```bash
az identity create -g <resourcegroup> -n <idname>
```

Example:

```bash
$ az identity create -g k8s-kv -n application-nginx1
```

Save the output, e.g.

```bash
{
  "clientId": "clientid-aaaa-bbbb-cccc-dddddddddddd",
  "clientSecretUrl": "https://control-eastus2.identity.azure.net/subscriptions/subscrip-tion-aaaa-bbbb-cccccccccccc/resourcegroups/k8s-kv/providers/Microsoft.ManagedIdentity/userAssignedIdentities/application-nginx1/credentials?tid=tenantid-aaaa-bbbb-cccc-dddddddddddd&oid=principa-lidx-aaaa-bbbb-cccccccccccc&aid=clientid-aaaa-bbbb-cccc-dddddddddddd",
  "id": "/subscriptions/subscrip-tion-aaaa-bbbb-cccccccccccc/resourcegroups/k8s-kv/providers/Microsoft.ManagedIdentity/userAssignedIdentities/application-nginx1",
  "location": "eastus2",
  "name": "application-nginx1",
  "principalId": "principa-lidx-aaaa-bbbb-cccccccccccc",
  "resourceGroup": "k8s-kv",
  "tags": {},
  "tenantId": "tenantid-aaaa-bbbb-cccc-dddddddddddd",
  "type": "Microsoft.ManagedIdentity/userAssignedIdentities"
}
```

## 4. Assign Permissions to new Identity for your keyvault

```bash
az role assignment create --role Reader --assignee <principalid> --scope /subscriptions/<subscriptionid>/resourcegroups/<resourcegroup>/providers/Microsoft.KeyVault/vaults/<keyvaultname>
```

Example:

```bash
$ az role assignment create --role Reader --assignee principa-lidx-aaaa-bbbb-cccccccccccc --scope /subscriptions/subscrip-tion-aaaa-bbbb-cccccccccccc/resourcegroups/k8s-kv/providers/Microsoft.KeyVault/vaults/k8s-kv1
```

Output, e.g.

```bash
{
  "canDelegate": null,
  "id": "/subscriptions/subscrip-tion-aaaa-bbbb-cccccccccccc/resourcegroups/k8s-kv/providers/Microsoft.KeyVault/vaults/k8s-kv1/providers/Microsoft.Authorization/roleAssignments/104df6be-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "name": "104df6be-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "principalId": "principa-lidx-aaaa-bbbb-cccccccccccc",
  "principalType": "ServicePrincipal",
  "resourceGroup": "k8s-kv",
  "roleDefinitionId": "/subscriptions/subscrip-tion-aaaa-bbbb-cccccccccccc/providers/Microsoft.Authorization/roleDefinitions/acdd72a7-3385-48ef-bd42-f606fba81ae7",
  "scope": "/subscriptions/subscrip-tion-aaaa-bbbb-cccccccccccc/resourcegroups/k8s-kv/providers/Microsoft.KeyVault/vaults/k8s-kv1",
  "type": "Microsoft.Authorization/roleAssignments"
}
```

## 5. Set policy to access keys in your keyvault

spn equals to `clientID` from the 'Create an Azure User Identity' step

```bash
az keyvault set-policy -n $KEYVAULT_NAME --key-permissions get --spn <YOUR AZURE USER IDENTITY CLIENT ID>
# set policy to access secrets in your keyvault
az keyvault set-policy -n $KEYVAULT_NAME --secret-permissions get --spn <YOUR AZURE USER IDENTITY CLIENT ID>
# set policy to access certs in your keyvault
az keyvault set-policy -n $KEYVAULT_NAME --certificate-permissions get --spn <YOUR AZURE USER IDENTITY CLIENT ID>
```

Example:

```bash
$ az keyvault set-policy -n k8s-kv1 --certificate-permissions get --spn clientid-aaaa-bbbb-cccc-dddddddddddd
```

## 6. Create your own SecretProviderClass Object

### 6.1 Get KV tenantId:

```bash
$ az keyvault show --resource-group k8s-kv --name k8s-kv1 | jq .properties.tenantId
```

### 6.2 Create SecretProviderClass:

```yml
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: azure-pi-kv-cert-nginx1
spec:
  provider: azure
  parameters:
    usePodIdentity: "true"
    keyvaultName: "k8s-kv1"
    cloudName: ""                                       # [OPTIONAL for Azure] if not provided, azure environment will default to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: testcert1
          objectType: cert                              # object types: secret, key or cert
          objectVersion: ""                             # [OPTIONAL] object versions, default to latest if empty
    tenantId: "tenantid-aaaa-bbbb-cccc-dddddddddddd"    # the tenant ID of the KeyVault
```

## 7. Create AzureIdentity

`resourceID: /subscriptions/<subid>/resourcegroups/<resourcegroup>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<idname>`
    `<subid>` – your subscriptionId
    `<idname>` - name of identity from the 'Create an Azure User Identity' step
`clientID`: the clientID from the 'Create an Azure User Identity' step

```yml
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentity
metadata:
  name: azure-pi-kv-cert-nginx1
spec:
  type: 0
  resourceID: /subscriptions/subscrip-tion-aaaa-bbbb-cccccccccccc/resourcegroups/k8s-kv/providers/Microsoft.ManagedIdentity/userAssignedIdentities/application-nginx1
  clientID: clientid-aaaa-bbbb-cccc-dddddddddddd
```

If you face the `error: unable to recognize "azure-pi-kv-cert-nginx1-AzureIdentity.yaml": no matches for kind "AzureIdentity" in version "aadpodidentity.k8s.io/v1"`
Your cluster doesn't have RBAC in place.

To setup RBAC:

```bash
kubectl apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment-rbac.yaml
```

## 8. create AzureIdentityBinding

`azureIdentity`: name of AzureIdentity from the 'Create AzureIdentity' step
`selector`: any label value to match in your app

```yml
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentityBinding
metadata:
  name: pod-aad-identity-binding
spec:
  azureIdentity: azure-pi-kv-cert-nginx1
  selector: app-nginx1
```

## 9. Create Depoloyment/Pod

```bash
metadata:
labels:
  aadpodidbinding: app-nginx1 # <AzureIdentityBinding Selector created from previous step>
```

`secretProviderClass`: the name of the class from the 'Create your own SecretProviderClass Object' step

```yml
kind: Pod
apiVersion: v1
metadata:
  name: nginx-secrets-pi
  labels:
    aadpodidbinding: "app-nginx1"
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
      - name: secrets-store01-inline
        mountPath: "/mnt/secrets-store"
        readOnly: true
  volumes:
    - name: secrets-store01-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-pi-kv-cert-nginx1"
```