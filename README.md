# Google Cloud Workload Identity for Jenkins

[![Badge: Google Cloud](https://img.shields.io/badge/Google%20Cloud-%234285F4.svg?logo=google-cloud&logoColor=white)](https://github.com/Cyclenerd/terraform-google-wif-jenkins#readme)
[![Badge: Terraform](https://img.shields.io/badge/Terraform-%235835CC.svg?logo=terraform&logoColor=white)](https://github.com/Cyclenerd/terraform-google-wif-jenkins#readme)
[![Badge: Jenkins](https://img.shields.io/badge/Jenkins-D24939.svg?logo=jenkins&logoColor=white)](https://github.com/Cyclenerd/terraform-google-wif-jenkins#readme)
[![Badge: CI](https://github.com/Cyclenerd/terraform-google-wif-jenkins/actions/workflows/ci.yml/badge.svg)](https://github.com/Cyclenerd/terraform-google-wif-jenkins/actions/workflows/ci.yml)
[![Badge: License](https://img.shields.io/github/license/cyclenerd/terraform-google-wif-jenkins)](https://github.com/Cyclenerd/terraform-google-wif-jenkins/blob/master/LICENSE)

This Terraform module creates a Workload Identity Pool and Provider for Jenkins with the [OpenID Connect Provider plugin](https://plugins.jenkins.io/oidc-provider/).

Service account keys are a security risk if compromised.
Avoid service account keys and instead use the [Workload Identity Federation](https://github.com/Cyclenerd/google-workload-identity-federation#readme).
For more information about Workload Identity Federation and how to best authenticate service accounts on Google Cloud, please see my GitHub repo [Cyclenerd/google-workload-identity-federation](https://github.com/Cyclenerd/google-workload-identity-federation#readme).

You are also welcome to take a look at the [comprehensive blueprint](https://github.com/Cyclenerd/gcp-jenkins) which describes in more detail how to securely connect Jenkins to Google Cloud Platform (GCP) using Workload Identity Federation.

> There are also a ready-to-use Terraform modules
> for [GitHub](https://github.com/Cyclenerd/terraform-google-wif-github#readme),
> [GitLab](https://github.com/Cyclenerd/terraform-google-wif-gitlab#readme)
> and [Bitbucket](https://github.com/Cyclenerd/terraform-google-wif-bitbucket#readme).

## Example

Create Workload Identity Pool and Provider:

```hcl
# Create Workload Identity Pool Provider for Jenkins
module "jenkins-wif" {
  source            = "Cyclenerd/wif-jenkins/google"
  version           = "~> 1.0"
  project_id        = var.project_id
  issuer_uri        = "https://jenkins.localhost"
  allowed_audiences = ["https://jenkins.localhost"]
  # Export of public OIDC JSON Web Key (JWK) file
  jwks_json_file    = "jenkins-jwk.json"
}

# Get the Workload Identity Pool Provider resource name for Jenkins configuration
output "jenkins-workload-identity-provider" {
  description = "The Workload Identity Provider resource name"
  value       = module.jenkins-wif.provider_name
}
```

Allow service account to login via Workload Identity Provider and limit login only from the Jenkins build `gcp-test` with URL `http://jenkins.localhost:2529/job/gcp-test/`:

```hcl
# Get existing service account for Jenkins
data "google_service_account" "jenkins" {
  project    = var.project_id
  account_id = "existing-account-for-jenkins"
}

# Allow service account to login via WIF and only from Jenkins build gcp-test
module "jenkins-service-account" {
  source     = "Cyclenerd/wif-service-account/google"
  version    = "~> 1.1"
  project_id = var.project_id
  pool_name  = module.jenkins-wif.pool_name
  account_id = data.google_service_account.jenkins.account_id
  subject    = "http://jenkins.localhost:2529/job/gcp-test/"
}
```

> Terraform module [`Cyclenerd/wif-service-account/google`](https://github.com/Cyclenerd/terraform-google-wif-service-account) is used.

👉 [**More examples**](https://github.com/Cyclenerd/terraform-google-wif-jenkins/tree/master/examples)

## Known Errors

### Whitespace Changes

The JSON file does not match the format stored in Google Cloud.
This means that every Terraform plan will result in a drift:

```text
jwks_json = jsonencode( # whitespace changes
```

Please see also the [GitHub Issue](https://github.com/hashicorp/terraform-provider-google/issues/23259).

To work around the issue, the JWK stored in Google Cloud can be downloaded and the JSON file must be overwritten manually.
It will then be in the same format as in Google Cloud.

Example to download the JWK:

```bash
gcloud iam workload-identity-pools providers describe "jenkins-oidc" \
    --workload-identity-pool="jenkins" \
    --location="global" \
    --project="YOUR-PROJECT" \
    --format="json" | jq -er '.oidc.jwksJson' | sed -e 's/\\"/"/g' -e 's/\\n/\n/g' | perl -pe 'chomp if eof' > "jwks.json"
```

## OIDC Token Attribute Mapping

> The attributes `attribute.sub` is used in the Terrform module [Cyclenerd/wif-service-account/google](https://github.com/Cyclenerd/terraform-google-wif-service-account).
> Please do not remove these attributes.

Default attribute mapping:

| Attribute        | Claim           | Description |
|------------------|-----------------|-------------|
| `google.subject` | `assertion.sub` | Subject
| `attribute.sub`  | `assertion.sub` | Defines the subject claim that is to be validated by the cloud provider. This setting is essential for making sure that access tokens are only allocated in a predictable way.
| `attribute.aud`  | `assertion.aud` | Intended audience for the token.
| `attribute.iss`  | `assertion.iss` | Issuer of the token.

<!-- BEGIN_TF_DOCS -->
## Providers

| Name | Version |
|------|---------|
| <a name="provider_google"></a> [google](#provider\_google) | >= 5.0.0 |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_allowed_audiences"></a> [allowed\_audiences](#input\_allowed\_audiences) | Workload Identity Pool Provider allowed audiences | `list(string)` | n/a | yes |
| <a name="input_attribute_condition"></a> [attribute\_condition](#input\_attribute\_condition) | (Optional) Workload Identity Pool Provider attribute condition expression | `string` | `null` | no |
| <a name="input_attribute_mapping"></a> [attribute\_mapping](#input\_attribute\_mapping) | Workload Identity Pool Provider attribute mapping | `map(string)` | <pre>{<br/>  "attribute.aud": "assertion.aud",<br/>  "attribute.iss": "assertion.iss",<br/>  "attribute.sub": "assertion.sub",<br/>  "google.subject": "assertion.sub"<br/>}</pre> | no |
| <a name="input_issuer_uri"></a> [issuer\_uri](#input\_issuer\_uri) | Workload Identity Pool Provider issuer URI | `string` | n/a | yes |
| <a name="input_jwks_json"></a> [jwks\_json](#input\_jwks\_json) | (Optional) OIDC JSON Web Key (JWK) in JSON String format. If not set, then the key is fetched from the `.well-known` path for the `issuer_uri` | `string` | `null` | no |
| <a name="input_jwks_json_file"></a> [jwks\_json\_file](#input\_jwks\_json\_file) | (Optional) OIDC JSON Web Key (JWK) file. If not set, then we use the `jwks_json` is used | `string` | `null` | no |
| <a name="input_pool_description"></a> [pool\_description](#input\_pool\_description) | Workload Identity Pool description | `string` | `"Workload Identity Pool for Jenkins (Terraform managed)"` | no |
| <a name="input_pool_disabled"></a> [pool\_disabled](#input\_pool\_disabled) | Workload Identity Pool disabled | `bool` | `false` | no |
| <a name="input_pool_display_name"></a> [pool\_display\_name](#input\_pool\_display\_name) | Workload Identity Pool display name | `string` | `"Jenkins"` | no |
| <a name="input_pool_id"></a> [pool\_id](#input\_pool\_id) | Workload Identity Pool ID | `string` | `"jenkins"` | no |
| <a name="input_project_id"></a> [project\_id](#input\_project\_id) | The ID of the project | `string` | n/a | yes |
| <a name="input_provider_description"></a> [provider\_description](#input\_provider\_description) | Workload Identity Pool Provider description | `string` | `"Workload Identity Pool Provider for Jenkins (Terraform managed)"` | no |
| <a name="input_provider_disabled"></a> [provider\_disabled](#input\_provider\_disabled) | Workload Identity Pool Provider disabled | `bool` | `false` | no |
| <a name="input_provider_display_name"></a> [provider\_display\_name](#input\_provider\_display\_name) | Workload Identity Pool Provider display name | `string` | `"Jenkins OIDC"` | no |
| <a name="input_provider_id"></a> [provider\_id](#input\_provider\_id) | Workload Identity Pool Provider ID | `string` | `"jenkins-oidc"` | no |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_pool_id"></a> [pool\_id](#output\_pool\_id) | Identifier for the pool |
| <a name="output_pool_name"></a> [pool\_name](#output\_pool\_name) | The resource name for the pool |
| <a name="output_pool_state"></a> [pool\_state](#output\_pool\_state) | State of the pool |
| <a name="output_provider_id"></a> [provider\_id](#output\_provider\_id) | Identifier for the provider |
| <a name="output_provider_name"></a> [provider\_name](#output\_provider\_name) | The resource name of the provider |
| <a name="output_provider_state"></a> [provider\_state](#output\_provider\_state) | State of the provider |
<!-- END_TF_DOCS -->

## License

All files in this repository are under the [Apache License, Version 2.0](LICENSE) unless noted otherwise.
