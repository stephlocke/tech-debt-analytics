# Security and Authentication within the project

## Deploying resources to Azure via CI/CD

To ensure the repository can securely deploy resources to Azure, the following steps are taken:

- [Optional but highly recommended] Create a [custom role](../infra/custom-deployer-role.json) in the subscription to allow deployment of the types of resources included in this solution with the least amount of privileges.
  - [Most recommended] You could provision the resource group first and associate the role only to that resource group to reduce the security perimeter of the role.
  - [Not recommended] Alternatively, you can assign contributor access to the subscription to the User Managed Identity when you create it.
- Create the [App Registration](#app-registration) that the Azure Functions will use to provide authentication.
- Create a [User Managed Identity](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/how-manage-user-assigned-managed-identities?pivots=identity-mi-methods-azcli) and assigned it to the custom role.
- Create an associated [Federated Credential](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation) for the User Managed Identity for the repository.
- Add the User Managed Identity Client ID within the [secrets for the repository](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions).
  - `AZURE_CLIENT_ID`: The client ID of the User Managed Identity.
  - `AZURE_TENANT_ID`: The tenant ID that the User Managed Identity exists in.
  - `AZURE_SUBSCRIPTION_ID`: The subscription ID where the resources are to be deployed.
- Add a login step within the CI/CD using the secrets associated with the User Managed Identity and the subscription you wish to deploy into.

```yaml
- name: Azure login
    uses: azure/login@v2
    with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

## Securing the solution

### Storage account

The Azure Storage Account and Blobs have several security measures applied:

1. **HTTPS Traffic Only**: The property `supportsHttpsTrafficOnly` is set to `true`. This means that all requests to this storage account must be made over HTTPS. Any requests made over HTTP will be rejected.

2. **OAuth Authentication**: The property `defaultToOAuthAuthentication` is set to `true`. This means that by default, OAuth is used for authentication. OAuth is a standard protocol that allows users to authenticate without sharing their password, providing a more secure way to control access.

3. **Hierarchical Namespace (HNS) Enabled**: The property `isHnsEnabled` is set to `true`. This means that the Azure Data Lake Storage Gen2 features, which include a hierarchical namespace and access control lists, are enabled for the storage account.

### Securing the Azure Function

The Azure Function is secured in several ways:

1. **HTTPS Only:** The property `httpsOnly` is set to `true`. This means that all requests to this function app must be made over HTTPS. Any requests made over HTTP will be rejected.

2. **Managed Identity:** The function app is assigned a [system-managed identity](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview). This allows the function app to authenticate to other Azure services using Azure Active Directory, without needing to store credentials in the code.

3. **Role Assignment:** The function app is given the role of 'Storage Blob Data Contributor' on the storage account. This means that the function app can read, write, and delete blobs in the storage account, but it does not have permission to manage the storage account itself. No other access is granted to the Function.

4. **Authentication Settings:** The function app is configured to require authentication (`requireAuthentication: true`). Unauthenticated requests will receive a 401 Unauthorized response (`unauthenticatedClientAction: 'Return401'`). The function app is configured to use Entra for authentication (`identityProviders.azureActiveDirectory.enabled: true`).

5. **Token Store:** The function app is configured to use a token store (`tokenStore.enabled: true`). This allows the function app to securely store and retrieve access tokens. The `tokenRefreshExtensionHours` is set to 72 hours, which means that the function app will automatically refresh access tokens that are within 72 hours of expiring.

6. **CORS:** The function app is configured to only allow CORS requests from 'https://github.com' (`siteConfig.cors.allowedOrigins: ['https://github.com']`). This can help to prevent cross-site request forgery attacks.

7. **Nonce Validation:** The `validateNonce` property is set to `true`, which means that the function app will validate the nonce value in the ID token that it receives from Azure AD. This can help to prevent replay attacks.

8. **Forward Proxy:** The convention is set to NoProxy, which means that the function app will not use a forward proxy for outgoing HTTP requests.

9. **Cookie Expiration:** The cookieExpiration is set to a fixed time of 8 hours (timeToExpiration: '08:00:00'). This means that authentication cookies will expire 8 hours after they are issued

10. **App registration:** The solution is restricted to logins being submitted to a specific client ID.

## How to securely work with the API

To successfully call a function hosted in this Function App, you need the following:

1. **Correct verb**: The function is expecting a PUT request.

2. **HTTPS only:** The function app is configured to only accept HTTPS requests.

3. **Authentication:** The function app is configured to require authentication. You need to include an access token in the Authorization header of the request.

### Steps to authenticate in a CI/CD process

To support authentication by the pipeline, you need to get the app registration's client ID plus the tenant ID and add them as [secrets in the repository](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions).

Then, within the pipeline you will need to login to Entra as the pipeline. The `--allow-no-subscriptions` flag is used to allow the pipeline to login even if though it won't be connected to any subscriptions.

```yaml
- name: Azure login
    uses: azure/login@v2
    with:
    # This is an app registration client ID associated with the shipper function.
    client-id: ${{ secrets.SHIPPER_CLIENT_ID }} 
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    allow-no-subscriptions: true
```

As the Function will need a Bearer token to authenticate, you will need to get a token from Entra that you can then pass as a header of the request. You can use the Azure CLI to get an access token.

```yaml
- name: Set environment variables
    run: |
    echo "ACCESS_TOKEN=$(az account get-access-token --query accessToken --resource  ${{ secrets.SHIPPER_CLIENT_ID }} -o tsv)" >> $GITHUB_ENV
```

You can then use the token in the Authorization header of the request.

```yaml
- name: Send report
    run: |
    response=$(curl -X ${{ env.FUNCTION_APP_VERB }} -v \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer ${{ env.ACCESS_TOKEN }}" \
    --data @${{ env.REPORT_JSON_FILENAME }} \
    "${{ env.SHIPPER_URL }}")
```

## App registration

The app registration is configured with the following settings:

- **Supported account types:** Accounts in this organizational directory only.
- **Allow public client flows:** No.

An example manifest for the app registration:

```json
{
 "id": "<guid>",
 "acceptMappedClaims": null,
 "accessTokenAcceptedVersion": null,
 "addIns": [],
 "allowPublicClient": null,
 "appId": "<guid>",
 "appRoles": [],
 "oauth2AllowUrlPathMatching": false,
 "createdDateTime": "2024-04-28T16:31:03Z",
 "description": null,
 "certification": null,
 "disabledByMicrosoftStatus": null,
 "groupMembershipClaims": null,
 "identifierUris": [],
 "informationalUrls": {
  "termsOfService": null,
  "support": null,
  "privacy": null,
  "marketing": null
 },
 "keyCredentials": [],
 "knownClientApplications": [],
 "logoUrl": null,
 "logoutUrl": null,
 "name": "Appcat DL Shipper",
 "notes": null,
 "oauth2AllowIdTokenImplicitFlow": false,
 "oauth2AllowImplicitFlow": false,
 "oauth2Permissions": [],
 "oauth2RequirePostResponse": false,
 "optionalClaims": null,
 "orgRestrictions": [],
 "parentalControlSettings": {
  "countriesBlockedForMinors": [],
  "legalAgeGroupRule": "Allow"
 },
 "passwordCredentials": [],
 "preAuthorizedApplications": [],
 "publisherDomain": "fdpo.onmicrosoft.com",
 "replyUrlsWithType": [],
 "requiredResourceAccess": [
  {
   "resourceAppId": "00000003-0000-0000-c000-000000000000",
   "resourceAccess": [
    {
     "id": "<guid>",
     "type": "Scope"
    }
   ]
  }
 ],
 "samlMetadataUrl": null,
 "signInUrl": null,
 "signInAudience": "AzureADMyOrg",
 "tags": [
  "apiConsumer",
  "backgroundProcess"
 ],
 "tokenEncryptionKeyId": null
}
```