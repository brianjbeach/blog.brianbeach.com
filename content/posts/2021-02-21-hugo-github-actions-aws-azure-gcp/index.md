---
layout: post
title: GitHub Actions for AWS, Azure and GCP
date: '2021-02-06'
author: Brian
tags: 
- Hugo
- AWS
- Azure
- GCP
- GitHub Actions
---

I'm abandoning the multi-cloud blog hosting model that I was using in favor of AWS Amplify to simplify TLS configuration. But I thought I should document the old approach a little further in case I ever go back to it. 

The build pipeline for my blog fails every once in a while. For example, there was an [issue with the Azure CLI](https://github.com/Azure/azure-cli/issues/16872) earlier this month. Each time that happens it takes me a few minutes to remember how the pipeline works. Therefore, I am documenting it quickly in this post. 

## Build

As I described in [this post]({{< ref "2019-12-30-multi-cloud-blog" >}}), my blog was hosted on AWS, Azure and GCP. There is a [GitHub Action that runs for each cloud provider ](https://github.com/brianjbeach/blog.brianbeach.com/tree/master/.github/workflows). Each script starts with the same three steps Checkout, Install, and Build. 

``` yaml
jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v1
      with:
        submodules: true

    - name: Install
      uses: peaceiris/actions-hugo@v2

    - name: Build
      run: |
        hugo version
        env HUGO_ENV="production" hugo --environment aws
```      

I'm using [peaceiris/actions-hugo@v2](https://github.com/peaceiris/actions-hugo) to build the hugo content. This was chosen at random. I have, on occasion, needed to specify a specific version of Hugo to avoid a bug. For example. 

``` yaml
      - name: Install
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.79.1'
```

Also, note the Hugo command line has a few options. The environment variable is needed to control the [Robots Meta Tag]({{< ref "2019-12-11-hugo-robots-meta-tag" >}}) and the environment flag [sets cloud provider in the page footer]({{< ref "2019-12-30-multi-cloud-blog" >}}).

``` bash
env HUGO_ENV="production" hugo --environment aws
```

The remainder of the GitHub Actions script is cloud provider specific. In each case, I am using the CLI to sync the latest static site to the cloud. 

## AWS

AWS uses a two step process. First I log in as an IAM user and then run `s3 sync`. 

``` yaml
    - name: Login
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1

    - name: Deploy
      run: |
        aws s3 sync public s3://blog.brianbeach.com --acl public-read --follow-symlinks --delete
```

I have an IAM user defined with permissions scoped to a single S3 bucket. The access and secret key for that user are stored as secrets in GitHub. 


## GCP

Google Cloud is nearly identical to AWS. There are distinct Login and Deploy steps.

``` yaml
    - name: Login
      uses: GoogleCloudPlatform/github-actions/setup-gcloud@master
      with:
        version: '270.0.0'
        service_account_key: ${{ secrets.GCP_SA_KEY }}
        export_default_credentials: true
        
    - name: Deploy
      run: |
        gsutil -m rsync -R public gs://blog.brianbeach.com
```

I have service account defined in GCP. The GCP_SA_KEY secret is the entire Key in JSON format. 

## Azure

Azure is a little different. Rather than creating a user, I am using the access keys for the storage account. Therefore, there is no distinct login step. 

``` yaml
    - name: Deploy
      uses: azure/CLI@v1
      with:
        inlineScript: |
          az storage blob sync -c '$web' -s "public" --account-name ${{ secrets.AZURE_ACCOUNT_NAME }} --account-key ${{ secrets.AZURE_ACCOUNT_KEY }}
```

The secrets are simply the name of the storage account and one of the two access keys. These are available on storage account page.