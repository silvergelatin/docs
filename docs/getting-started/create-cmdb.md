---
sidebar_label: Creating a CMDB
title: Creating a CMDB
---

The Configuration Management Database (CMDB) is the heart of hamlet. It contains the configuration, infrastructure code and state of your hamlet deployed infrastructure.

CMDBs use a file based structure with a strict directory layout that hamlet uses to build its configuration using an overlay style approach. We won't go to much into how all of this works at this stage, if you'd like to know more about it head to the [layer anatomy in-depth guide](/docs/in-depth/layers/)

### Generating the CMDB

We can start a CMDB using the cli

1. Create a new directory that will be used for the CMDB. This directory contains your infrastructure and should be treated as code

    :::info
    This directory is generally added to your source control provider to share the configuration with your team or CI deployments
    :::

    ```bash
    mkdir ~/hamlet_hello/mycmdb
    ```

1. Change into the directory and create the root.json file. This file tells hamlet where the top of the CMDB file structure is

    ```bash
    cd ~/hamlet_hello/mycmdb
    echo '{}' > root.json
    ```

1. Now that we have the root defined lets add our Tenant. The tenant represents your overall hamlet deployment. This can be a single application with a single cloud account or all of your applications and all of their cloud accounts. Hamlet allows you to define your infrastructure in a standardised way based on your deployment requirements.

    We can create a tenant using the hamlet cli. It will create the files and folder structure that we need and ask some questions to fill in the gaps

    Inside your CMDB directory run the following command

    ```bash
    hamlet generate tenant-cmdb
    ```

    You will be prompted for a details that describe your tenancy

    ```bash
    [?] tenant id: acmeinc
    [?] tenant name [acmeinc]:
    [?] default region [ap-southeast-2]:
    [?] audit log expiry days [2555]:
    [?] audit log offline days [90]:
    ```

    Enter the details as requested and a new directory will be created in your cmdb called **accounts**

    ```bash
    tree  ~/hamlet_hello/mycmdb
    ```

    ```bash
    |-- accounts
    |   -- acmeinc
    |       |-- ipaddressgroups.json
    |       |-- tenant.json
    |-- root.json
    ```

1. Once you have your tenant CMDB we now need to create an account to host our infrastructure in. Accounts hold the configuration of your cloud account or subscription and can be shared across different applications as you need to.

    change into the tenant cmdb directory

    ```bash
    cd ~/hamlet_hello/mycmdb/accounts
    ```

    run the the following to create your account cmdb

    ```bash
    hamlet generate account-cmdb
    ```

    Enter the required details, the ones without defaults, in the command and a new account directory will be created in your cmdb based on the account name

    :::info
    If you are planning on completing our AWS tutorial enter aws when asked for the provider type
    :::

    :::info
    For now you can use a random Id for the provider id field. This is the Id of your cloud provider account that we will deploy into.
    :::

    ```bash
    [?] account id: acct01
    [?] account name [acct01]:
    [?] account seed [te2k1xv5om]:
    [?] provider type (aws, azure) [aws]: aws
    [?] provider id: 1234567890
    ```

    ```bash
    tree  ~/hamlet_hello/mycmdb
    ```

    ```bash
    |-- accounts
    |   |-- acct01
    |   |   |-- config
    |   |   |   |-- account.json
    |   |   |   |-- settings
    |   |   |       |-- shared
    |   |   |           |-- settings.json
    |   |   |-- infrastructure
    |   |       |-- operations
    |   |           |-- shared
    |   |               |-- credentials.json
    |   |-- acmeinc
    |       |-- ipaddressgroups.json
    |       |-- tenant.json
    |-- root.json
    ```

1. Now that the overall hosting environment has been defined we can create a product. Products represent applications deployed through their lifecycle

    change into the root directory of your CMDB

    ```bash
    cd ~/hamlet_hello/mycmdb
    ```

    run the following to create your product cmdb

    ```bash
    hamlet generate product-cmdb
    ```

    Enter the required details, which won't have default values

    :::info
    leave the dns zone section empty to remove the requirement to setup DNS zones
    :::

    ```bash
    [?] product id: myapp
    [?] dns zone []:
    [?] product name [myapp]:
    [?] environment id [int]:
    [?] environment name [integration]:
    [?] segment id [default]:
    [?] segment name [default]:
    ```

## Ready To Build

Now that you have your base CMDB setup you can move on to our deployment guides which cover how to use the CMDB with real world examples

- [AWS Serverless](aws-serverless/index.md) - Deploy an API and Web interface using serverless techniques in Amazon Web Services
