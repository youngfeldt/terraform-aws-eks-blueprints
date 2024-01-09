.ONESHELL:
	SHELL := /bin/bash
	.PHONY: help
ENV = $(shell basename $$(dirname $$(pwd)))
TF_VAR_environment = $(ENV)
TF_VAR_current_region = $(shell aws configure get region)
PROJECT = $(shell basename $$(pwd))
BUCKET_KEY = "states/$(PROJECT)"
BUCKET_NAME = "enclave-terraform-state"
TF_VAR_bucket_name = $(BUCKET_NAME)
BUCKET_REGION = "$(TF_VAR_current_region)"
TFVARS_FILE="terraform.tfvars"
DYNAMO_TABLE="tf_backend_db"


help:
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

showvars: ## Displays variables at this point in tree.
	@echo "REGION      = $(TF_VAR_current_region)"
	@echo "BUCKET      = $(BUCKET_NAME)"
	@echo "BUCKET_KEY  = $(BUCKET_KEY)"

tfvars:
	@if [ ! -f terraform.tfvars ]; then\
		echo "no terraform.tfvars, will touch one";\
		touch terraform.tfvars;\
		fi

initial: init ## Copies local state to backend state
	@rm -rf .terraform
	@terraform init -force-copy

init: showvars tfvars
	@rm -rf .terraform/terraform.tfstate
	@echo "Running init, using s3 bucket $(BUCKET_NAME)/$(BUCKET_KEY) for storage"
	@echo "# remote state is generated on each make init run" > terraform.tf
	@echo "terraform {"                                      >> terraform.tf
	@echo "  backend \"s3\"  {"                              >> terraform.tf
	@echo "    bucket      = \"$(BUCKET_NAME)\""             >> terraform.tf
	@echo "    acl         = \"bucket-owner-full-control\""  >> terraform.tf
	@echo "    encrypt     = \"false\""                      >> terraform.tf
	@echo "    key         = \"$(BUCKET_KEY)\""              >> terraform.tf
	@echo "    region      = \"$(BUCKET_REGION)\""           >> terraform.tf
#	@echo "    dynamodb_table  = \"${DYNAMO_TABLE}\""        >> terraform.tf
	@echo "  }"                                              >> terraform.tf
	@echo "}"                                                >> terraform.tf
	@export TF_VAR_project="$(PROJECT)"	
	@terraform init \
		-backend-config="region=$(BUCKET_REGION)" \
		-backend-config="bucket=$(BUCKET_NAME)" \
		-backend-config="key=$(BUCKET_KEY)/terraform.tfstate"

update:
	@terraform get -update=true 1>/dev/null

plan: init update ## Shows what a plan looks like for applying a specific resource
	@terraform plan \
		-input=true \
		-refresh=true \
		-out .plan.out \
		-var-file=$(TFVARS_FILE)

plan-destroy: init update ## Creates a destruction plan.
	@terraform plan \
		-input=false \
		-refresh=true \
		-module-depth=-1 \
		-destroy \
		-var-file=$(TFVARS_FILE)


destroy: init update plan-destroy ## Destroys everything. There is a prompt before destruction.
	@terraform destroy \
		-var-file=$(TFVARS_FILE)


show: init
	@terraform show -module-depth=-1

graph: ## Output the `dot` graph of all the built Terraform resources
	@rm -f graph.png
	@terraform graph -draw-cycles -module-depth=-1 | dot -Tpng > graph.png
# 	@shotwell graph.png

apply: init update ## Apply builds/changes resources. You should ALWAYS run a plan first.
	@terraform apply \
		-input=true \
		-refresh=true \
		-var-file=$(TFVARS_FILE)

auto-apply: init 
	@terraform apply \
		-auto-approve \
		-refresh=true \
		-var-file=$(TFVARS_FILE)

apply-target: init update ## Apply builds/changes resources. You should ALWAYS run a plan first.
	@if [ ! -z ${run_module} ]; then\
		echo "need run_module=xxx";\
		exit 1;\
		fi
	@terraform apply \
		-input=true \
		-refresh=true \
		-target=module.${run_module} \
		-var-file=$(TFVARS_FILE)

output: init update ## Display all outputs from the remote state file.
	@echo "Example to type for the module: MODULE=module.rds.aws_route53_record.rds-master"
	@echo
	@if [ -z $(MODULE) ]; then\
		terraform output;\
		else\
		terraform output -module=$(MODULE);\
		fi
