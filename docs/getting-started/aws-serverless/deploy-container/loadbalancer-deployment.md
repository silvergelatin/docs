---
sidebar_label: API Load Balancer
title: API Load Balancer
---

Following on from the container deployment, this guide will walk through exposing our container to the internet through a load balancer

Before walking through the concepts we will first run the deployment and then go through what was created

:::info
If you haven't already, create a CMDB using the [create CMDB guide](../../create-cmdb.md)
When we talk about the CMDB it will be based on the naming used in the setup guide
:::

:::warning
For this deployment to work please complete the [Hello Status API Deployment](./hamlet-hello-api.md) guide first and then come back to this one
:::

1. Change into the integration default segment directory to set the context

    ```bash
    # Make sure we are in our CMDB
    cd ~/hamlet_hello/mycmdb

    # Change into the default segment for the integration environment
    cd myapp/config/solutionsv2/integration/default/
    ```

    open the `segment.json` in your code editor

1. Add the following to the segment file

    ```json
    {
        "Tiers" : {
            "elb" : {
                "Components" : {
                    "apicdn" : {
                        "cdn" : {
                            "Instances" : {
                                "default" : {
                                    "deployment:Unit" : "apicdn",
                                    "deployment:Priority" : 150
                                }
                            },
                            "Pages": {
                                "Root": "",
                                "Denied": "",
                                "Error": "",
                                "NotFound": ""
                            },
                            "Routes" : {
                                "default" : {
                                    "Origin" : {
                                        "Link" : {
                                            "Tier" : "elb",
                                            "Component" : "apilb",
                                            "PortMapping" : "http"
                                        }
                                    },
                                    "PathPattern" : "_default"
                                }
                            }
                        }
                    },
                    "apilb" : {
                        "lb" : {
                            "Instances" : {
                                "default" : {
                                    "deployment:Unit" : "apilb"
                                }
                            },
                            "Engine" : "application",
                            "PortMappings" : {
                                "http" : {
                                    "IPAddressGroups" : [ "_global" ],
                                    "Forward" : {},
                                    "Mapping" : "httpflask"
                                }
                            }
                        }
                    }
                }
            }
        },
        "Ports" : {
            "flask": {
                "Port": 8000,
                "Protocol": "HTTP",
                "IPProtocol": "tcp",
                "HealthCheck": {
                    "Path": "/",
                    "HealthyThreshold": "3",
                    "UnhealthyThreshold": "5",
                    "Interval": "30",
                    "Timeout": "5"
                }
            }
        },
        "PortMappings" : {
            "httpflask" : {
                "Source": "http",
                "Destination": "flask"
            }
        }
    }
    ```

    This includes:

    - a public facing load balancer that listens for http requests on port 80 and forward them to port 8000 on the backend services
    - a CDN which will provide an HTTPS based endpoint and forward requests to the load balancer

1. When we define load balancers we don't define the backends that it will forward requests to, instead the backend services register with the load balancer. So lets add that to the solution as well

    ```json
    {
        "Services" : {
            "helloapi" : {
                "Containers" : {
                    "api": {
                        "Ports" : {
                            "flask" : {
                                "LB" : {
                                    "Tier" : "elb",
                                    "Component" : "apilb",
                                    "PortMapping" : "http"
                                }
                            }
                        }
                    }
                }
            }
        }
    }
    ```

1. Update the deployments to create the load balancer and configure the containers to register with the load balancer

    ```bash
    hamlet --account acct01 deploy run-deployments
    ```

## Testing the API

Once the deployments have completed lets test our api and see if its working. To do that we need to know the URL of the load balancer.

To find this, Lets look at some new hamlet commands which give us details on our occurrences. We've been talking about components and subcomponents, occurrences are the individual instances of components and subcomponents. The occurrence represents a collection of resources which are deployed as a specific function. They are what hamlet uses as the key representation of our deployments.

So first up lets see what occurrences exist in our solution

```bash
hamlet --account acct01 component list-occurrences
```

```terminal
| TierId   | ComponentId   | Name                                      | Type               |
|----------|---------------|-------------------------------------------|--------------------|
| elb      | apicdn        | elb-apicdn-cdn                            | cdn                |
| elb      | apicdn        | elb-apicdn-default-cdnroute               | cdnroute           |
| elb      | apilb         | elb-apilb-lb                              | lb                 |
| elb      | apilb         | elb-apilb-http-lbport                     | lbport             |
| app      | ecshost       | application-ecshost-ecs                   | ecs                |
| app      | ecshost       | application-ecshost-helloapi-service      | service            |
| mgmt     | baseline      | management-baseline-baseline              | baseline           |
| mgmt     | baseline      | management-baseline-opsdata-baselinedata  | baselinedata       |
| mgmt     | baseline      | management-baseline-appdata-baselinedata  | baselinedata       |
| mgmt     | baseline      | management-baseline-ssh-baselinekey       | baselinekey        |
| mgmt     | baseline      | management-baseline-cmk-baselinekey       | baselinekey        |
| mgmt     | baseline      | management-baseline-oai-baselinekey       | baselinekey        |
| mgmt     | vpc           | management-vpc-network                    | network            |
| mgmt     | vpc           | management-vpc-internal-networkroute      | networkroute       |
| mgmt     | vpc           | management-vpc-external-networkroute      | networkroute       |
| mgmt     | vpc           | management-vpc-open-networkacl            | networkacl         |
| mgmt     | igw           | management-igw-gateway                    | gateway            |
| mgmt     | igw           | management-igw-default-gatewaydestination | gatewaydestination |
```

The names should look familiar in the componentId column these are the keys under the Components sections that we've been adding to the segment.json file. Each of the components listed here have multiple occurrences with different types, each type is a different collection of resources that implement a function within the component. The cdn is a good example of this, you can have one cdn endpoint, but it can have different routes based on path you ask for in your http request.

So to find the url for the http port lets look at the details of the lbport.

```bash
hamlet --account acct01 component describe-occurrence -n elb-apicdn-cdn attributes
```

```terminal
| Key             | Value                                 |
|-----------------|---------------------------------------|
| FQDN            | abc1234567.cloudfront.net             |
| DISTRIBUTION_ID | ABC123ABC12                           |
| INTERNAL_FQDN   | abc1234567.cloudfront.net             |
| URL             | https://abc1234567.cloudfront.net     |
```

And here we get a collection of details about the load balancer port, including the URL.

Now that we have the URL lets see if our API is working

```bash
curl https://abc1234567.cloudfront.net
```

```terminal
{
    "Greeting": "Hello!",
    "Location": "nowhere"
}
```

There is our API returning our greeting over HTTPS and served by a CDN

## Reference Data

As part of this guide we added some new sections to the CMDB, Ports and PortMappings. These are known as reference data within hamlet, they are reusable configurations that can be reused across your solution. Ports and PortMappings are examples of the Reference Data Types that are available, and under each type you have instances of the reference. So we added a new port called flask along with a new port mapping called httpflask.

You might notice that there is another port mentioned in the port mapping, http. This does exist in the Reference Data collection but has been provided as part of the references available in the engine. Hamlet provides a collection of in-built reference data that you can replace or add to as you need.

## Links

When we updated our service to register with the load balancer we needed to reference the load balancer. This was done using a link, A link is an object structure that is used in solutions to establish relationships between occurrences. Links have the following properties

- Tier: the id of the tier the component belongs to
- Component: The id of the component you want to link to
- `<Sub Component Type>`: the type of a  subcomponent you want to link to as the key with the value as the Id of the subcomponent
- Instance: an optional id of the instance for the component to link to, by default the source component instance id is used
- Version: an optional id of the version for the component to link to, by default the source component version id is used
- Role: The role used is used to apply permissions between different components, for example a component can specify write access to another component and its permissions will be deployed with the appropriate permissions.

For the container we used the Tier, Component and Sub Component Type properties to establish a link between the container in the api service and the apilb

Next up is configuring the location where our api is, nowhere isn't to friendly to say hello from.
