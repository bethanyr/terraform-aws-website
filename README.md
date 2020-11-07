# terraform-aws-website
Creates a public or private website behind a cloudfront distribution, with SSL enabled, including support for multiple domain names (e.g. www.example.com as well as example.com). CORS is also configured for you. Cognito hosted UI is put in front of it.

The website files are hosted in an S3 bucket which is also created by the module.

# Usage
```hcl-terraform
module "website" {
    source = "bwindsor/website/aws"
    
    deployment_name = "tf-my-project"
    website_dir = "${path.root}/website_files"
    additional_files = { "config.yaml" = <<EOF
a: 1
b: 2
EOF
  }
    hosted_zone_name = "example.com"
    custom_domain = "example.com"
    alternative_custom_domains = ["www.example.com"]
    template_file_vars = { api_url = "api.mysite.com" }
    is_spa = false
    csp_allow_default = ["api.mysite.com"]
    csp_allow_style = ["'unsafe-inline'"]
    csp_allow_img = ["data:"]
    csp_allow_font = []
    csp_allow_frame = []
    mime_types = {}

    is_private = true
    
    # These settings are only used when is_private is true
    create_cognito_pool = true
    refresh_auth_path = "/refreshauth"
    logout_path = "/"
    parse_auth_path = "/parseauth"
    refresh_token_validity_days = 3650
    oauth_scopes = ["openid"]
    
    # This setting is only required when is_private is true and create_cognito_pool is true
    auth_domain_prefix = "mydomain"
    
    # This block is required when is_private is true and create_cognito_pool is false. Otherwise it is ignored.
    cognito = {
        user_pool_arn = aws_cognito_user_pool.mypool.arn
        client_id = aws_cognito_user_pool_client.myclient.id
        auth_domain = "https://mydomain.auth.eu-west-1.amazoncognito.com"
    }
}
```

### Inputs
Ensure environment variables `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` are set.

* **deployment_name** A unique string to use for this module to make sure resources do not clash with others
* **website_dir** A folder containing all the files for your website. The contents of this folder, including all subfolders, will be stored in an S3 website and served as your website
* **additional_files** A mapping from file name (in S3) to file contents. For each (key,value) pair, a file will be created in S3 with the given key, with contents given by value
* **hosted_zone_name** The name of the hosted zone in Route53 where the SSL certificates will be created
* **custom_domain** The primary domain name to use for the website
* **alternative_custom_domains** A list of any alternative domain names. Typically this would just contain the same as *custom_domain* but prefixed by `www.`
* **template_file_vars** A mapping from substitution variable name to value. Any files inside `website_dir` which end in `.template` will be processed by Terraform's template provider, passing these variables for substitution. The file will have the `.template` suffix removed when uploaded to S3
* **is_spa** If your website is a single page application (SPA), this sets up the cloudfront redirects such that whenever an item is not found, the file `index.html` is returned instead
* **csp_allow_default** List of default domains to include in the Content Security Policy. Typically you would list the URL of your API here if your pages access that. Always includes `'self'`
* **csp_allow_style** List of places to allow CSP to load styles from. Always includes `'self'`
* **csp_allow_img** List of places to allow CSP to load images from. Always includes `'self'`
* **csp_allow_font** List of places to allow CSP to load fonts from. Always includes `'self'`
* **csp_allow_frame** List of places to allow CSP to load iframes from. Always includes `'self'`
* **mime_types** Map from file extension to MIME type. Defaults are provided, but you will need to provide any unusual extensions with a MIME type
* **is_private** Boolean, default true. Whether to make the site private (behind Cognito)
* **create_cognito_pool** Boolean, default true. Whether to create a Cognito pool for authentication. If false, a `cognito` configuration must be provided
* **refresh_auth_path** Path relative to `custom_domain` to redirect to when a token refresh is required
* **logout_path** Path relative to `custom_domain` to redirect to after logging out
* **parse_auth_path** Path relative to `custom_domain` to redirect to upon successful authentication
* **refresh_token_validity_days** Time until the refresh token expires and the user will be required to log in again
* **oauth_scopes** List of auth scopes to grant (or which are granted, if a Cognito pool is created externally). Options include phone, email, profile, openid, aws.cognito.signin.user.admin
* **auth_domain_prefix** The first part of the hosted UI login domain, as in https://{AUTH_DOMAIN_PREFIX}.auth.region.amazoncognito.com
* **cognito** Configuration block for an existing user pool. Required when `create_cognito_pool` is false
    * **user_pool_arn** User pool ARN of an existing user pool
    * **client_id** Client ID of an existing user pool client
    * **auth_domain** Domain name for the existing hosted UI. Could be in the format https://{AUTH_DOMAIN_PREFIX}.auth.region.amazoncognito.com or could be a custom domain

### Outputs
* **url** The URL on which the home page of the website can be reached
* **alternate_urls** Alternate URLs which also point to the same home page as *url* does