---
title: "Cloudquery - Getting Started"
date: 2022-02-08T08:06:15Z
---

# What is CloudQuery?

CloudQuery is a powerful tool that allows us to extract, normalize, expose and export the configuration and metadata of infrastructure resources deployed in different cloud computing providers.

The result of the operations performed by CloudQUery is stored in a Postgres database, allowing us to write SQL queries to facilitate monitoring, governance and security.

To do so, CloudQuery abstracts different APIs that allow us to define security, governance, cost and SQL compliance policies.

# Why use CloudQuery?

CloudQuery gives unprecedented power and visibility to your cloud infrastructure and SaaS applications in a normalized way that is accessible with SQL in order to do security and compliance, cloud inventory, asset managementc and auditing tasks.

# Getting started.

## Download and install.

You can download the precompiled binary from releases, or using HomeBrew.

### OSX.

```
breaking_pitt@Converge~ curl -L https://github.com/cloudquery/cloudquery/releases/latest/download/cloudquery_darwin_x86_64 -o cloudquery
breaking_pitt@Converge~ chmod a+x cloudquery
```

### Install with HomeBrew.

```
breaking_pitt@Converge~ brew install cloudquery/tap/cloudquery
```

### Check CloudQuery install.

You can check if the binary has been installed correctly, running the following command:

```
breaking_pitt@Converge~ cloudquery --version
cloudquery version 0.20.2
```

## Setup PostgreSQL database.

As mentioned at the beginning of this article, the information retrieved by CloudQuery is exported to a Postgres database, so we are going to need one. The easiest way is to use a Docker container.

```
breaking_pitt@Converge~ mkdir -p cloudquery/test/db
breaking_pitt@Converge~ docker run -d \
                           --name cloudquery-db \
                           -p 5432:5432 \
                           -e POSTGRES_PASSWORD=cl09dqu3r13 \
                           -e PGDATA=/var/lib/postgresql/data/pgdata \
                           -v ~/cloudquery/test/db:/var/lib/postgresql/data \
                           postgres
```

# Running CloudQuery.

Now that we have the CloudQuery tool installed on your machine, create a directory and initialize the CloudqQuery configuration inside it, with the following command:

```
breaking_pitt@Converge~ mkdir -p cloudquery/test && cd .cloudquery/test
breaking_pitt@Converge~ cloudquery --no-telemetry init aws
```

The init command will generate a config.hcl file that will describe which cloud provider we want to use and which resources we want CloudQuery to ETL. If we do not modify anything CloudQuery will bring all the resources of all the subscriptions to which have access. 

By default, CloudQuery connects to the PostgreSQL database that is defined in the config.hcl's connection section, as we've changed the default database name in our PostgreSQL server set up, we must edit this section to configure the location and credentials of your postgres database.

```
cloudquery {
  ...
  ...

  connection {
    dsn = "host=localhost user=postgres password=cl09dqu3r13 database=cloudquery-db port=5432"
  }
}
```

Now we have our project ready to retrieve the information of our subscriptions.

## Authenticating with AWS.

CloudQuery needs to be authenticated with your AWS account in order to gather all the information about your cloud setup. There are multiple ways to authenticate with AWS, by default, CloudQuery respects the AWS credential provider chain. This means that CloudQuery will follow the following priorities when attempting to authenticate:

- Enviroment variables -> AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, and AWS_SESSION_TOKEN.
- Configuration file -> Credentials and config files stored in ~/.aws folder.
- IAM roles for AWS compute resources..

## Gather and dump data into PostgreSQL with CloudQuery.

Once we have customized our ```config.hcl``` and we are authenticated with AWS, run the following command to fetch the resources.

```
breaking_pitt@Converge~ cloudquery fetch
```

Once the process is finished, we can see that you have a bunch of tables generated. We can connect to our PostgreSQL server to check if CloudQuery found information about our cloud infrastructure.

```
breaking_pitt@Converge~ psql "postgres://postgres:cl09dqu3r13@localhost:5432/cloudquery-db?sslmode=disable"
cloudquery-db=# \d
postgres=# \d
                                     List of relations
 Schema |                              Name                              | Type  |  Owner
--------+----------------------------------------------------------------+-------+----------
 public | aws_access_analyzer_analyzer_finding_sources                   | table | postgres
 public | aws_access_analyzer_analyzer_findings                          | table | postgres
 public | aws_access_analyzer_analyzers                                  | table | postgres
 public | aws_accounts                                                   | table | postgres
 public | aws_acm_certificates                                           | table | postgres
 public | aws_apigateway_api_keys                                        | table | postgres
 public | aws_apigateway_client_certificates                             | table | postgres
 public | aws_apigateway_domain_name_base_path_mappings                  | table | postgres
 public | aws_apigateway_domain_names                                    | table | postgres
 public | aws_apigateway_rest_api_authorizers                            | table | postgres
 public | aws_apigateway_rest_api_deployments                            | table | postgres
 public | aws_apigateway_rest_api_documentation_parts                    | table | postgres
 public | aws_apigateway_rest_api_documentation_versions                 | table | postgres
 public | aws_apigateway_rest_api_gateway_responses                      | table | postgres
 public | aws_apigateway_rest_api_models                                 | table | postgres
 public | aws_apigateway_rest_api_request_validators                     | table | postgres
 public | aws_apigateway_rest_api_resources                              | table | postgres
 public | aws_apigateway_rest_api_stages                                 | table | postgres
 public | aws_apigateway_rest_apis                                       | table | postgres
 ...
```

## Query dumped data.

Now that the ```cloudquery fetch``` command dumped all the information about the infrastructure from our AWS account into our PostgreSQL server, we can start to get information about our infrastructure, for example:

```
cloudquery-db=# \d aws_ec2_vpcs;
                 Table "public.aws_ec2_vpcs"
      Column      |  Type   | Collation | Nullable | Default
------------------+---------+-----------+----------+---------
 cq_id            | uuid    |           | not null |
 cq_meta          | jsonb   |           |          |
 account_id       | text    |           | not null |
 region           | text    |           |          |
 arn              | text    |           |          |
 cidr_block       | text    |           |          |
 dhcp_options_id  | text    |           |          |
 instance_tenancy | text    |           |          |
 is_default       | boolean |           |          |
 owner_id         | text    |           |          |
 state            | text    |           |          |
 tags             | jsonb   |           |          |
 id               | text    |           | not null |
Indexes:
    "aws_ec2_vpcs_pk" PRIMARY KEY, btree (account_id, id)
    "aws_ec2_vpcs_cq_id_key" UNIQUE CONSTRAINT, btree (cq_id)
Referenced by:
    TABLE "aws_ec2_vpc_cidr_block_association_sets" CONSTRAINT "aws_ec2_vpc_cidr_block_association_sets_vpc_cq_id_fkey" FOREIGN KEY (vpc_cq_id) REFERENCES aws_ec2_vpcs(cq_id) ON DELETE CASCADE
    TABLE "aws_ec2_vpc_ipv6_cidr_block_association_sets" CONSTRAINT "aws_ec2_vpc_ipv6_cidr_block_association_sets_vpc_cq_id_fkey" FOREIGN KEY (vpc_cq_id) REFERENCES aws_ec2_vpcs(cq_id) ON DELETE CASCADE
```

And run our queries:

```
cloudquery-db=# SELECT account_id, region, arn, cidr_block FROM aws_ec2_vpcs WHERE region='eu-west-1' AND is_default='t';
  account_id  |  region   |                         arn                         |  cidr_block
--------------+-----------+-----------------------------------------------------+---------------
 635811601642 | eu-west-1 | arn:aws:ec2:eu-west-1:635811601642:vpc/vpc-13db3b77 | 172.31.0.0/16
```


