# tf-null-label

Terraform module designed to generate consistent label names and tags for resources. Use `tf-null-label` to implement a strict naming convention.

A label follows the following convention: `{namespace}-{environment}-{stage}-{name}-{attributes}`. The delimiter (e.g. `-`) is interchangeable.
The label items are all optional. So if you perfer the term `stage` to `environment` you can exclude environment and the label `id` will look like `{namespace}-{stage}-{name}-{attributes}`.
If attributes are excluded but `stage` and `environment` are included, `id` will look like `{namespace}-{environment}-{stage}-{name}`

It's recommended to use one `tf-null-label` module for every unique resource of a given resource type.
For example, if you have 10 instances, there should be 10 different labels.
However, if you have multiple different kinds of resources (e.g. instances, security groups, file systems, and elastic ips), then they can all share the same label assuming they are logically related.


## Usage

### Simple Example

```hcl
module "eg_prod_bastion_label" {
  source     = "../../tf-module/tf-null-label"
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
  source     = "../../tf-module/tf-null-label"
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
  source     = "../../tf-module/tf-null-label"
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
# tf-null-label example #
################################
module "label" {
  source    = "../../tf-module/tf-null-label"
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
  source      = "../../tf-module/tf-null-label"
  namespace   = "Domain"
  environment = "UAT"
  stage       = "build"
  name        = "Winston Churchroom"
  attributes  = ["fire", "water", "earth", "air"]
  delimiter   = "-"

  label_order = ["name", "environment", "stage", "attributes"]

  tags = {
    "City"        = "Dublin"
    "Environment" = "Private"
  }
}

module "label2" {
  source    = "../../tf-module/tf-null-label"
  context   = "${module.label1.context}"
  name      = "Charlie"
  stage     = "test"
  delimiter = "+"

  tags = {
    "City"        = "London"
    "Environment" = "Public"
  }
}

module "label3" {
  source    = "../../tf-module/tf-null-label"
  name      = "Starfish"
  stage     = "release"
  context   = "${module.label1.context}"
  delimiter = "."

  tags = {
    "Eat"    = "Carrot"
    "Animal" = "Rabbit"
  }
}
```

This creates label outputs like this:

```hcl
label1 = {
  attributes = [fire water earth air]
  id = winstonchurchroom-uat-build-fire-water-earth-air
  name = winstonchurchroom
  namespace = domain
  stage = build
}
label1_context = {
  attributes = [fire water earth air]
  delimiter = [-]
  environment = [uat]
  label_order = [name environment stage attributes]
  name = [winstonchurchroom]
  namespace = [domain]
  stage = [build]
  tags_keys = [City Environment Name Namespace Stage]
  tags_values = [Dublin Private winstonchurchroom-uat-build-fire-water-earth-air domain build]
}
label1_tags = {
  City = Dublin
  Environment = Private
  Name = winstonchurchroom-uat-build-fire-water-earth-air
  Namespace = domain
  Stage = build
}
label2 = {
  attributes = [fire water earth air]
  id = charlie+uat+test+fire+water+earth+air
  name = charlie
  namespace = domain
  stage = test
}
label2_context = {
  attributes = [fire water earth air]
  delimiter = [+]
  environment = [uat]
  label_order = [name environment stage attributes]
  name = [charlie]
  namespace = [domain]
  stage = [test]
  tags_keys = [City Environment Name Namespace Stage]
  tags_values = [London Public charlie+uat+test+fire+water+earth+air domain test]
}
label2_tags = {
  City = London
  Environment = Public
  Name = charlie+uat+test+fire+water+earth+air
  Namespace = domain
  Stage = test
}
label3 = {
  attributes = [fire water earth air]
  id = starfish.uat.release.fire.water.earth.air
  name = starfish
  namespace = domain
  stage = release
}
label3_context = {
  attributes = [fire water earth air]
  delimiter = [.]
  environment = [uat]
  label_order = [name environment stage attributes]
  name = [starfish]
  namespace = [domain]
  stage = [release]
  tags_keys = [Animal City Eat Environment Name Namespace Stage]
  tags_values = [Rabbit Dublin Carrot uat starfish.uat.release.fire.water.earth.air domain release]
}
label3_tags = {
  Animal = Rabbit
  City = Dublin
  Eat = Carrot
  Environment = uat
  Name = starfish.uat.release.fire.water.earth.air
  Namespace = domain
  Stage = release
}


## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|:----:|:-----:|:-----:|
| additional_tag_map | Additional tags for appending to each tag map | map | `<map>` | no |
| attributes | Additional attributes (e.g. `1`) | list | `<list>` | no |
| context | Default context to use for passing state between label invocations | map | `<map>` | no |
| delimiter | Delimiter to be used between `namespace`, `environment`, `stage`, `name` and `attributes` | string | `-` | no |
| enabled | Set to false to prevent the module from creating any resources | string | `true` | no |
| environment | Environment, e.g. 'prod', 'staging', 'dev', 'pre-prod', 'UAT' | string | `` | no |
| label_order | The naming order of the id output and Name tag | list | `<list>` | no |
| name | Solution name, e.g. 'app' or 'jenkins' | string | `` | no |
| namespace | Namespace, which could be your organization name or abbreviation, e.g. 'eg' or 'domain' | string | `` | no |
| stage | Stage, e.g. 'prod', 'staging', 'dev', OR 'source', 'build', 'test', 'deploy', 'release' | string | `` | no |
| tags | Additional tags (e.g. `map('BusinessUnit','XYZ')` | map | `<map>` | no |

## Outputs

| Name | Description |
|------|-------------|
| attributes | List of attributes |
| context | Context of this module to pass to other label modules |
| delimiter | Delimiter between `namespace`, `environment`, `stage`, `name` and `attributes` |
| environment | Normalized environment |
| id | Disambiguated ID |
| label_order | The naming order of the id output and Name tag |
| name | Normalized name |
| namespace | Normalized namespace |
| stage | Normalized stage |
| tags | Normalized Tag map |
| tags_as_list_of_maps | Additional tags as a list of maps, which can be used in several AWS resources |
