<!-- This file was automatically generated by the `build-harness`. Make all changes to `README.yaml` and run `make readme` to rebuild this file. -->

[![Cloud Posse](https://cloudposse.com/logo-300x69.svg)](https://cloudposse.com)

# terraform-null-label [![Build Status](https://travis-ci.org/cloudposse/terraform-null-label.svg?branch=master)](https://travis-ci.org/cloudposse/terraform-null-label-cloudtrail) [![Latest Release](https://img.shields.io/github/release/cloudposse/terraform-null-label.svg)](https://github.com/cloudposse/terraform-null-label/releases/latest) [![Slack Community](https://slack.cloudposse.com/badge.svg)](https://slack.cloudposse.com)


Terraform module designed to generate consistent label names and tags for resources. Use `terraform-null-label` to implement a strict naming convention.

A label follows the following convention: `{namespace}-{environment}-{stage}-{name}-{attributes}`. The delimiter (e.g. `-`) is interchangeable.
The label items are all optional. So if you perfer the term `stage` to `environment` you can exclude environment and the label `id` will look like `{namespace}-{stage}-{name}-{attributes}`.
If attributes are excluded but `stage` and `environment` are included, `id` will look like `{namespace}-{environment}-{stage}-{name}`

It's recommended to use one `terraform-null-label` module for every unique resource of a given resource type.
For example, if you have 10 instances, there should be 10 different labels.
However, if you have multiple different kinds of resources (e.g. instances, security groups, file systems, and elastic ips), then they can all share the same label assuming they are logically related.

All [Cloud Posse modules](https://github.com/cloudposse?utf8=%E2%9C%93&q=terraform-&type=&language=) use this module to ensure resources can be instantiated multiple times within an account and without conflict.

**NOTE:** The `null` refers to the primary Terraform [provider](https://www.terraform.io/docs/providers/null/index.html) used in this module.


---

This project is part of our comprehensive ["SweetOps"](https://docs.cloudposse.com) approach towards DevOps. 


It's 100% Open Source and licensed under the [APACHE2](LICENSE).










## Usage

### Simple Example

```hcl
module "eg_prod_bastion_label" {
  source     = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  namespace  = "eg"
  stage      = "prod"
  name       = "bastion"
  attributes = ["public"]
  delimiter  = "-"
  tags       = "${map("BusinessUnit", "XYZ", "Snapshot", "true")}"
}
```

This will create an `id` with the value of `eg-prod-bastion-public` because when generating `id`, the default order is `namespace`, `environment`, `stage`,  `name`, `attributes`
(you can override it by using the `label_order` variable, see [Advanced Example 3](#advanced-example-3)).

Now reference the label when creating an instance:

```hcl
resource "aws_instance" "eg_prod_bastion_public" {
  instance_type = "t1.micro"
  tags          = "${module.eg_prod_bastion_label.tags}"
}
```

Or define a security group:

```hcl
resource "aws_security_group" "eg_prod_bastion_public" {
  vpc_id = "${var.vpc_id}"
  name   = "${module.eg_prod_bastion_label.id}"
  tags   = "${module.eg_prod_bastion_label.tags}"
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```


### Advanced Example

Here is a more complex example with two instances using two different labels. Note how efficiently the tags are defined for both the instance and the security group.

```hcl
module "eg_prod_bastion_abc_label" {
  source     = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  namespace  = "eg"
  stage      = "prod"
  name       = "bastion"
  attributes = ["abc"]
  delimiter  = "-"
  tags       = "${map("BusinessUnit", "ABC")}"
}

resource "aws_security_group" "eg_prod_bastion_abc" {
  name = "${module.eg_prod_bastion_abc_label.id}"
  tags = "${module.eg_prod_bastion_abc_label.tags}"
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "eg_prod_bastion_abc" {
   instance_type          = "t1.micro"
   tags                   = "${module.eg_prod_bastion_abc_label.tags}"
   vpc_security_group_ids = ["${aws_security_group.eg_prod_bastion_abc.id}"]
}

module "eg_prod_bastion_xyz_label" {
  source     = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  namespace  = "eg"
  stage      = "prod"
  name       = "bastion"
  attributes = ["xyz"]
  delimiter  = "-"
  tags       = "${map("BusinessUnit", "XYZ")}"
}

resource "aws_security_group" "eg_prod_bastion_xyz" {
  name = "${module.eg_prod_bastion_xyz_label.id}"
  tags = "${module.eg_prod_bastion_xyz_label.tags}"
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "eg_prod_bastion_xyz" {
   instance_type          = "t1.micro"
   tags                   = "${module.eg_prod_bastion_xyz_label.tags}"
   vpc_security_group_ids = ["${aws_security_group.eg_prod_bastion_xyz.id}"]
}
```

### Advanced Example 2

Here is a more complex example with an autoscaling group that has a different tagging schema than other resources and requires its tags to be in this format, which this module can generate:

```hcl
tags = [
    {
        key = Name,
        propagate_at_launch = 1,
        value = namespace-stage-name
    },
    {
        key = Namespace,
        propagate_at_launch = 1,
        value = namespace
    },
    {
        key = Stage,
        propagate_at_launch = 1,
        value = stage
    }
]
```

Autoscaling group using propagating tagging below (full example: [autoscalinggroup](examples/autoscalinggroup/main.tf))

```hcl
################################
# terraform-null-label example #
################################
module "label" {
  source    = "../../"
  namespace = "cp"
  stage     = "prod"
  name      = "app"

  tags = {
    BusinessUnit = "Finance"
    ManagedBy    = "Terraform"
  }

  additional_tag_map = {
    propagate_at_launch = "true"
  }
}

#######################
# Launch template     #
#######################
resource "aws_launch_template" "default" {
  # terraform-null-label example used here: Set template name prefix
  name_prefix                           = "${module.label.id}-"
  image_id                              = "${data.aws_ami.amazon_linux.id}"
  instance_type                         = "t2.micro"
  instance_initiated_shutdown_behavior  = "terminate"

  vpc_security_group_ids                = ["${data.aws_security_group.default.id}"]

  monitoring {
    enabled                             = false
  }
  # terraform-null-label example used here: Set tags on volumes
  tag_specifications {
    resource_type                       = "volume"
    tags                                = "${module.label.tags}"
  }
}

######################
# Autoscaling group  #
######################
resource "aws_autoscaling_group" "default" {
  # terraform-null-label example used here: Set ASG name prefix
  name_prefix                           = "${module.label.id}-"
  vpc_zone_identifier                   = ["${data.aws_subnet_ids.all.ids}"]
  max_size                              = "1"
  min_size                              = "1"
  desired_capacity                      = "1"

  launch_template = {
    id                                  = "${aws_launch_template.default.id}"
    version                             = "$$Latest"
  }

  # terraform-null-label example used here: Set tags on ASG and EC2 Servers
  tags                                  = ["${module.label.tags_as_list_of_maps}"]
}
```

### Advanced Example 3

See [complete example](./examples/complete)

This example shows how you can pass the `context` output of one label module to the next label_module,
allowing you to create one label that has the base set of values, and then creating every extra label
as a derivative of that.

```hcl
module "label1" {
  source      = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  namespace   = "CloudPosse"
  environment = "UAT"
  stage       = "build"
  name        = "Winston Churchroom"
  attributes  = ["fire", "water", "earth", "air"]

  label_order = ["name", "environment", "stage", "attributes"]

  tags = {
    "City"        = "Dublin"
    "Environment" = "Private"
  }
}

module "label2" {
  source  = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  context = "${module.label1.context}"
  name    = "Charlie"
  stage   = "test"

  tags = {
    "City"        = "London"
    "Environment" = "Public"
  }
}

module "label3" {
  source = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  name   = "Starfish"
  stage  = "release"

  tags = {
    "Eat"    = "Carrot"
    "Animal" = "Rabbit"
  }
}
```

This creates label outputs like this:

```hcl
label1 = {
  attributes = fire-water-earth-air
  id = winstonchurchroom-uat-build-fire-water-earth-air
  name = winstonchurchroom
  namespace = cloudposse
  stage = build
}
label1_context = {
  attributes = [fire-water-earth-air]
  delimiter = [-]
  environment = [uat]
  label_order = [name environment stage attributes]
  name = [winstonchurchroom]
  namespace = [cloudposse]
  stage = [build]
  tags_keys = [City Environment Name Namespace Stage]
  tags_values = [Dublin Private winstonchurchroom-uat-build-fire-water-earth-air cloudposse build]
}
label1_tags = {
  City = Dublin
  Environment = Private
  Name = winstonchurchroom-uat-build-fire-water-earth-air
  Namespace = cloudposse
  Stage = build
}
label2 = {
  attributes = fire-water-earth-air
  id = charlie-uat-test-fire-water-earth-air
  name = charlie
  namespace = cloudposse
  stage = test
}
label2_context = {
  attributes = [fire-water-earth-air]
  delimiter = [-]
  environment = [uat]
  label_order = [name environment stage attributes]
  name = [charlie]
  namespace = [cloudposse]
  stage = [test]
  tags_keys = [City Environment Name Namespace Stage]
  tags_values = [London Public charlie-uat-test-fire-water-earth-air cloudposse test]
}
label2_tags = {
  City = London
  Environment = Public
  Name = charlie-uat-test-fire-water-earth-air
  Namespace = cloudposse
  Stage = test
}
label3 = {
  attributes =
  id = release-starfish
  name = starfish
  namespace =
  stage = release
}
label3_context = {
  attributes = []
  delimiter = [-]
  environment = []
  label_order = [namespace environment stage name attributes]
  name = [starfish]
  namespace = []
  stage = [release]
  tags_keys = [Animal Eat Environment Name Namespace Stage]
  tags_values = [Rabbit Carrot  release-starfish  release]
}
label3_tags = {
  Animal = Rabbit
  Eat = Carrot
  Environment =
  Name = release-starfish
  Namespace =
  Stage = release
}  ```






## Makefile Targets
```
Available targets:

  help                                Help screen
  help/all                            Display help for all targets
  help/short                          This help short screen
  lint                                Lint terraform code

```

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|:----:|:-----:|:-----:|
| additional_tag_map | Additional tags for appending to each tag map | map | `<map>` | no |
| attributes | Additional attributes (e.g. `1`) | list | `<list>` | no |
| context | Default context to use for passing state between label invocations | map | `<map>` | no |
| delimiter | Delimiter to be used between `name`, `namespace`, `stage`, etc. | string | `-` | no |
| enabled | Set to false to prevent the module from creating any resources | string | `true` | no |
| environment | Environment, e.g. 'prod', 'staging', 'dev', 'pre-prod', 'UAT' | string | `` | no |
| label_order | The naming order of the id output and Name tag | list | `<list>` | no |
| name | Solution name, e.g. 'app' or 'jenkins' | string | `` | no |
| namespace | Namespace, which could be your organization name, e.g. 'eg' or 'cp' | string | `` | no |
| stage | Stage, e.g. 'prod', 'staging', 'dev', OR 'source', 'build', 'test', 'deploy', 'release' | string | `` | no |
| tags | Additional tags (e.g. `map('BusinessUnit','XYZ')` | map | `<map>` | no |

## Outputs

| Name | Description |
|------|-------------|
| attributes | Normalized attributes |
| context | Context of this module to pass between other modules |
| delimiter | Delimiter used in label ID |
| environment | Normalized environment |
| id | Disambiguated ID |
| label_order | The naming order of the id output and Name tag |
| name | Normalized name |
| namespace | Normalized namespace |
| stage | Normalized stage |
| tags | Normalized Tag map |
| tags_as_list_of_maps | Additional tags as a list of maps, which can be used in several AWS resources |




## Related Projects

Check out these related projects.

- [terraform-terraform-label](https://github.com/cloudposse/terraform-terraform-label) - Terraform Module to define a consistent naming convention by (namespace, environment, stage, name, [attributes])



## Help

**Got a question?**

File a GitHub [issue](https://github.com/cloudposse/terraform-null-label/issues), send us an [email][email] or join our [Slack Community][slack].

## Commercial Support

Work directly with our team of DevOps experts via email, slack, and video conferencing. 

We provide [*commercial support*][commercial_support] for all of our [Open Source][github] projects. As a *Dedicated Support* customer, you have access to our team of subject matter experts at a fraction of the cost of a full-time engineer. 

[![E-Mail](https://img.shields.io/badge/email-hello@cloudposse.com-blue.svg)](mailto:hello@cloudposse.com)

- **Questions.** We'll use a Shared Slack channel between your team and ours.
- **Troubleshooting.** We'll help you triage why things aren't working.
- **Code Reviews.** We'll review your Pull Requests and provide constructive feedback.
- **Bug Fixes.** We'll rapidly work to fix any bugs in our projects.
- **Build New Terraform Modules.** We'll develop original modules to provision infrastructure.
- **Cloud Architecture.** We'll assist with your cloud strategy and design.
- **Implementation.** We'll provide hands-on support to implement our reference architectures. 


## Community Forum

Get access to our [Open Source Community Forum][slack] on Slack. It's **FREE** to join for everyone! Our "SweetOps" community is where you get to talk with others who share a similar vision for how to rollout and manage infrastructure. This is the best place to talk shop, ask questions, solicit feedback, and work together as a community to build *sweet* infrastructure.

## Contributing

### Bug Reports & Feature Requests

Please use the [issue tracker](https://github.com/cloudposse/terraform-null-label/issues) to report any bugs or file feature requests.

### Developing

If you are interested in being a contributor and want to get involved in developing this project or [help out](https://github.com/orgs/cloudposse/projects/3) with our other projects, we would love to hear from you! Shoot us an [email](mailto:hello@cloudposse.com).

In general, PRs are welcome. We follow the typical "fork-and-pull" Git workflow.

 1. **Fork** the repo on GitHub
 2. **Clone** the project to your own machine
 3. **Commit** changes to your own branch
 4. **Push** your work back up to your fork
 5. Submit a **Pull Request** so that we can review your changes

**NOTE:** Be sure to merge the latest changes from "upstream" before making a pull request!


## Copyright

Copyright © 2017-2018 [Cloud Posse, LLC](https://cloudposse.com)



## License 

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) 

See [LICENSE](LICENSE) for full details.

    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.









## Trademarks

All other trademarks referenced herein are the property of their respective owners.

## About

This project is maintained and funded by [Cloud Posse, LLC][website]. Like it? Please let us know at <hello@cloudposse.com>

[![Cloud Posse](https://cloudposse.com/logo-300x69.svg)](https://cloudposse.com)

We're a [DevOps Professional Services][hire] company based in Los Angeles, CA. We love [Open Source Software](https://github.com/cloudposse/)!

We offer paid support on all of our projects.  

Check out [our other projects][github], [apply for a job][jobs], or [hire us][hire] to help with your cloud strategy and implementation.

  [docs]: https://docs.cloudposse.com/
  [website]: https://cloudposse.com/
  [github]: https://github.com/cloudposse/
  [commercial_support]: https://github.com/orgs/cloudposse/projects
  [jobs]: https://cloudposse.com/jobs/
  [hire]: https://cloudposse.com/contact/
  [slack]: https://slack.cloudposse.com/
  [linkedin]: https://www.linkedin.com/company/cloudposse
  [twitter]: https://twitter.com/cloudposse/
  [email]: mailto:hello@cloudposse.com


### Contributors

|  [![Erik Osterman][osterman_avatar]][osterman_homepage]<br/>[Erik Osterman][osterman_homepage] | [![Andriy Knysh][aknysh_avatar]][aknysh_homepage]<br/>[Andriy Knysh][aknysh_homepage] | [![Igor Rodionov][goruha_avatar]][goruha_homepage]<br/>[Igor Rodionov][goruha_homepage] | [![Sergey Vasilyev][s2504s_avatar]][s2504s_homepage]<br/>[Sergey Vasilyev][s2504s_homepage] | [![Michael Pereira][MichaelPereira_avatar]][MichaelPereira_homepage]<br/>[Michael Pereira][MichaelPereira_homepage] | [![Jamie Nelson][Jamie-BitFlight_avatar]][Jamie-BitFlight_homepage]<br/>[Jamie Nelson][Jamie-BitFlight_homepage] | [![Vladimir][SweetOps_avatar]][SweetOps_homepage]<br/>[Vladimir][SweetOps_homepage] | [![Daren Desjardins][darend_avatar]][darend_homepage]<br/>[Daren Desjardins][darend_homepage] |
|---|---|---|---|---|---|---|---|

  [osterman_homepage]: https://github.com/osterman
  [osterman_avatar]: https://github.com/osterman.png?size=150
  [aknysh_homepage]: https://github.com/aknysh
  [aknysh_avatar]: https://github.com/aknysh.png?size=150
  [goruha_homepage]: https://github.com/goruha
  [goruha_avatar]: https://github.com/goruha.png?size=150
  [s2504s_homepage]: https://github.com/s2504s
  [s2504s_avatar]: https://github.com/s2504s.png?size=150
  [MichaelPereira_homepage]: https://github.com/MichaelPereira
  [MichaelPereira_avatar]: https://github.com/MichaelPereira.png?size=150
  [Jamie-BitFlight_homepage]: https://github.com/Jamie-BitFlight
  [Jamie-BitFlight_avatar]: https://github.com/Jamie-BitFlight.png?size=150
  [SweetOps_homepage]: https://github.com/SweetOps
  [SweetOps_avatar]: https://github.com/SweetOps.png?size=150
  [darend_homepage]: https://github.com/darend
  [darend_avatar]: https://github.com/darend.png?size=150


