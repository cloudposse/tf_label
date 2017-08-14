# tf_label

Terraform module designed to generate consistent label names and tags for resources. Use `tf_label` to implement a strict naming convention. 

A label follows the following convention: `{namespace}-{stage}-{name}-{attributes}`. The delimiter (e.g. `-`) is interchangable. 

All [Cloud Posse modules](https://github.com/cloudposse?utf8=%E2%9C%93&q=tf_&type=&language=) use this module to ensure resources can be instantiated multiple times within an account without conflict.

## Usage

### Simple Example

Include this repository as a module in your existing terraform code:

```
module "eg_prod_bastion_label" {
  source          = "git::https://github.com/cloudposse/tf_label.git?ref=tags/0.2.0"
  namespace       = "eg"
  stage           = "prod"
  name            = "bastion"
  attributes      = ["public"]
  delimiter       = "-"
  tags            = "${map("BusinessUnit", "XYZ", "Snapshot", "true")}"
}
```

This will create an `id` with the value of `eg-prod-bastion-public`. 

Now reference the label when creating an instance (for example):
```
resource "aws_instance" "eg_prod_bastion_public" {
  instance_type = "t1.micro"
  tags          = "${module.eg_prod_bastion_label.tags}"
}
```

Or define a security group:
```
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

```
module "eg_prod_bastion_abc_label" {
  source          = "git::https://github.com/cloudposse/tf_label.git?ref=tags/0.2.0"
  namespace       = "eg"
  stage           = "prod"
  name            = "bastion"
  attributes      = ["abc]
  delimiter       = "-"
  tags            = "${map("BusinessUnit", "ABC")}"
}

resource "aws_security_group" "eg_prod_bastion_abc" {
  name          = "module.eg_prod_bastion_abc_label.id"
  tags          = "${module.eg_prod_bastion_abc_label.tags}"

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
   vpc_security_group_ids = [aws_security_group.eg_prod_bastion_abc.id]
} 

module "eg_prod_bastion_xyz_label" {
  source          = "git::https://github.com/cloudposse/tf_label.git?ref=tags/0.2.0"
  namespace       = "egample"
  stage           = "prod"
  name            = "bastion"
  attributes      = ["xyz"]
  delimiter       = "-"
  tags            = "${map("BusinessUnit", "XYZ")}"
}

resource "aws_security_group" "eg_prod_bastion_xyz" {
  name          = "module.eg_prod_bastion_xyz_label.id"
  tags          = "${module.eg_prod_bastion_xyz_label.tags}"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "eg_prod_bastion_xyz" {
   instance_type          = "t1.micro"
   tags                   = "${module.xyz_bastion_label.tags}"
   vpc_security_group_ids = [aws_security_group.eg_prod_bastion_xyz.id]
}
```


## Variables

|  Name                        |  Default       |  Description                                              | Required |
|:----------------------------:|:--------------:|:--------------------------------------------------------:|:--------:|
| namespace                    | ``             | Namespace (e.g. `cp` or `cloudposse`)                    | Yes      |
| stage                        | ``             | Stage (e.g. `prod`, `dev`, `staging`                     | Yes      |
| name                         | ``             | Name  (e.g. `bastion` or `db`)                           | Yes      | 
| attributes                   | []             | Additional attributes (e.g. `policy` or `role`)          | No       | 
| tags                         | {}             | Additional tags  (e.g. `map("BusinessUnit","XYZ")`       | No       |

## Outputs

| Name              | Decription            |
|:-----------------:|:---------------------:|
| id                | Disambiguated ID      |
| name              | Normalized name       |
| namespace         | Normalized namespace  |
| stage             | Normalized stage      |
| attributes        | Normalized attributes |
| tags              | Normalized Tag map    |

