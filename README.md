## Amazon ECR "Login" Action for GitHub Actions

Logs in the local Docker client to one or more Amazon ECR Private registries or an Amazon ECR Public registry.

**Table of Contents**

<!-- toc -->

- [Example of Usage](#example-of-usage)
- [Credentials and Region](#credentials-and-region)
- [Permissions](#permissions)
- [License Summary](#license-summary)
- [Security Disclosures](#security-disclosures)

<!-- tocstop -->

## Example of Usage

Logging in to Amazon ECR Private, then building and pushing a Docker image:
```yaml
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build, tag, and push docker image to Amazon ECR
      env:
        REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        REPOSITORY: my-ecr-repo
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $REGISTRY/$REPOSITORY:$IMAGE_TAG .
        docker push $REGISTRY/$REPOSITORY:$IMAGE_TAG
```

Logging in to Amazon ECR Public, then building and pushing a Docker image:
```yaml
    - name: Login to Amazon ECR Public
      id: login-ecr-public
      uses: aws-actions/amazon-ecr-login@v1
      with:
        registry-type: 'public'

    - name: Build, tag, and push docker image to Amazon ECR Public
      env:
        REGISTRY: ${{ steps.login-ecr-public.outputs.registry }}
        REGISTRY_ALIAS: my-ecr-public-registry-alias
        REPOSITORY: my-ecr-public-repo
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $REGISTRY/$REGISTRY_ALIAS/$REPOSITORY:$IMAGE_TAG .
        docker push $REGISTRY/$REGISTRY_ALIAS/$REPOSITORY:$IMAGE_TAG
```

Logging in to Amazon ECR Private, then packaging and pushing a Helm chart:
```yaml
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - name: Package and push helm chart to Amazon ECR
      env:
        REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        REPOSITORY: my-ecr-repo
      run: |
        helm package $REPOSITORY
        helm push $REPOSITORY-0.1.0.tgz oci://$REGISTRY
```

Logging in to Amazon ECR Public, then packaging and pushing a Helm chart:
```yaml
    - name: Login to Amazon ECR Public
      id: login-ecr-public
      uses: aws-actions/amazon-ecr-login@v1
      with:
        registry-type: 'public'

    - name: Package and push helm chart to Amazon ECR Public
      env:
        REGISTRY: ${{ steps.login-ecr-public.outputs.registry }}
        REGISTRY_ALIAS: my-ecr-public-registry-alias
        REPOSITORY: my-ecr-public-repo
      run: |
        helm package $REPOSITORY
        helm push $REPOSITORY-0.1.0.tgz oci://$REGISTRY/$REGISTRY_ALIAS
```

Helm uses the same credential store as Docker. So Helm can authenticate with the same credentials that you use for Docker.

See [action.yml](action.yml) for the full documentation for this action's inputs and outputs.

## Credentials and Region

### AWS Credentials

This action relies on the [default behavior of the AWS SDK for Javascript](https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/setting-credentials-node.html) to determine AWS credentials and region.  Use [the `aws-actions/configure-aws-credentials` action](https://github.com/aws-actions/configure-aws-credentials) to configure the GitHub Actions environment with a role using GitHub's OIDC provider and your desired region.

```yaml
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        role-to-assume: arn:aws:iam::123456789012:role/my-github-actions-role
        aws-region: us-east-1

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
```

We recommend following [Amazon IAM best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) when using AWS services in GitHub Actions workflows, including:
* [Assume an IAM role](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#delegate-using-roles) to receive temporary credentials. See the [Sample IAM Role CloudFormation Template](https://github.com/aws-actions/configure-aws-credentials#sample-iam-role-cloudformation-template) in the `aws-actions/configure-aws-credentials` action to get an example.
* [Grant least privilege](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege) to the IAM role used in GitHub Actions workflows.  Grant only the permissions required to perform the actions in your GitHub Actions workflows.  See the Permissions section below for the permissions required by this action.
* [Monitor the activity](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#keep-a-log) of the IAM role used in GitHub Actions workflows.

### Docker Credentials
After the authentication, you can access the docker username and password via Action outputs using the following format:
- Registry URI for ECR Private: `123456789012.dkr.ecr.aws-region-1.amazonaws.com`
- Registry URI for ECR Public: `public.ecr.aws`

If using ECR Private:
- Docker username output: `docker_username_123456789012_dkr_ecr_aws_region_1_amazonaws_com`
- Docker password output: `docker_password_123456789012_dkr_ecr_aws_region_1_amazonaws_com`

If using ECR Public:
- Docker username output: `docker_username_public_ecr_aws`
- Docker password output: `docker_password_public_ecr_aws`

To push Helm charts, you can also login through Docker. By default, Helm can authenticate with the same credentials that you use for Docker.

## Permissions

### ECR Private

To see how and where to implement the permissions below, see the [IAM section in the Amazon ECR User Guide](https://docs.aws.amazon.com/AmazonECR/latest/userguide/security-iam.html).

This action requires the following minimum set of permissions to login to ECR Private:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GetAuthorizationToken",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    }
  ]
}
```

Docker commands in your GitHub Actions workflow, like `docker pull` and `docker push`, may require additional permissions attached to the credentials used by this action.

The following minimum permissions are required for pulling an image from an ECR Private repository:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPull",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
      ],
      "Resource": "arn:aws:ecr:us-east-1:123456789012:repository/my-ecr-repo"
    }
  ]
}
```

The following minimum permissions are required for pushing and pulling images in an ECR Private repository:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPushPull",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:GetDownloadUrlForLayer",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:UploadLayerPart"
      ],
      "Resource": "arn:aws:ecr:us-east-1:123456789012:repository/my-ecr-repo"
    }
  ]
}
```

### ECR Public

To see how and where to implement the permissions below, see the [IAM section in the Amazon ECR Public User Guide](https://docs.aws.amazon.com/AmazonECR/latest/public/security-iam.html).


This action requires the following minimum set of permissions to login to ECR Public:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GetAuthorizationToken",
      "Effect": "Allow",
      "Action": [
        "ecr-public:GetAuthorizationToken"
      ],
      "Resource": "*"
    }
  ]
}
```

Docker commands in your GitHub Actions workflow, like `docker push`, may require additional permissions attached to the credentials used by this action. There are no permissions needed for pulling images from ECR Public.

The following minimum permissions are required for pushing an image to an ECR Public repository:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPush",
      "Effect": "Allow",
      "Action": [
        "ecr-public:BatchCheckLayerAvailability",
        "ecr-public:CompleteLayerUpload",
        "ecr-public:InitiateLayerUpload",
        "ecr-public:PutImage",
        "ecr-public:UploadLayerPart"
      ],
      "Resource": "arn:aws:ecr-public:us-east-1:123456789012:repository/my-ecr-public-repo"
    }
  ]
}
```

## License Summary

This code is made available under the MIT license.

## Security Disclosures

If you would like to report a potential security issue in this project, please do not create a GitHub issue.  Instead, please follow the instructions [here](https://aws.amazon.com/security/vulnerability-reporting/) or [email AWS security directly](mailto:aws-security@amazon.com).
