.PHONY: init test
.DEFAULT_GOAL := help
SHELL=/usr/bin/env bash -e -o pipefail

CFN_TEMPLATE := ${CFN_TEMPLATE}
CFN_STACKNAME := ${CFN_STACKNAME}
CFN_SERVICEROLE := ${CFN_SERVICEROLE}
CFN_TEMPLATEDIR = $(shell dirname $(CFN_TEMPLATE))


TMP_MSG := ".tmpmsg"
TMP_RET := ".tmpret"

install: ## install cfn check tool ## -
	pip install awscli cfn-lint 
	gem install cfn-nag

check-dependency = $(if $(shell command -v $(1)),,$(error Make sure $(1) is installed))

init:
	@$(call check-dependency,aws)
	@$(call check-dependency,jq)
	@$(call check-dependency,cfn-lint)
	@$(call check-dependency,cfn_nag)

cfn-lint:init
	@echo "=== cfn-lint ==="
	@find $(CFN_TEMPLATEDIR) -type f \( -name "*.yaml" -or -name "*.yml" \) | \
		while read f; do ( cfn-lint -t $$f && echo "√ $$f") || exit 1 ; done

cfn-nag:cfn-lint
	@echo "=== cfn-nag ==="
	@cfn_nag_scan --input-path=$(CFN_TEMPLATEDIR) --allow-suppression

test:cfn-nag

cfn-validate:
	@echo "=== cfn-validate ==="
	@aws cloudformation validate-template \
		--template-body file://$(CFN_TEMPLATE)

listchangeset:cfn-validate ## list stack changesets ## -
	@$$((aws cloudformation deploy		\
		--template-file $(CFN_TEMPLATE) \
		--stack-name $(CFN_STACKNAME)	\
		--no-execute-changeset			\
		--capabilities CAPABILITY_NAMED_IAM \
		1> $(TMP_MSG)  2>&1 ) ;echo $${PIPESTATUS[0]} > $(TMP_RET))
	@# Check return code and msg. If no changeset, cfn deploy will return 255 with msg  "No changes to deploy"
	@if [ $$(cat $(TMP_RET)) -eq 0 ]; then \
		echo "=== List ChangeSet ==="; \
		$$(cat $(TMP_MSG) | grep "aws cloudformation") | jq -r '.Changes[]|.ResourceChange.Action + "\t" + .ResourceChange.ResourceType + "     \t" + .ResourceChange.LogicalResourceId + "   \t["+.ResourceChange.PhysicalResourceId + "]"';  \
	elif [ $$(cat $(TMP_RET)) -eq 255 ] && grep -qF "No changes to deploy" $(TMP_MSG) ; then \
		cat $(TMP_MSG); \
		exit 0; \
	else \
		echo "=== Error !!! ==="; \
		cat $(TMP_MSG); \
		exit $$(cat $(TMP_RET)); \
	fi 

# cfn deploy
deploy:cfn-validate ## deploy cfn ## -
	aws cloudformation deploy      \
		--template-file $(CFN_TEMPLATE) \
		--stack-name $(CFN_STACKNAME)	\
		--no-fail-on-empty-changeset	\
		--capabilities CAPABILITY_NAMED_IAM \
		--role-arn $(CFN_SERVICEROLE)	\
		;echo $${PIPESTATUS[0]} > $(TMP_RET)
	@if [ $$(cat $(TMP_RET)) -eq 0 ]; then \
		echo "Deploy done."; \
	else \
		echo "Deploy Error !!! "; \
		exit $$(cat $(TMP_RET)); \
	fi 	

help: ## print this message ## -
	@echo "The cfn operation by make."
	@echo ""
	@printf "\033[36m%-30s\033[0m %-50s %s\n" "[Command]" "[Description]" "[Example]"
	@grep -E '^[/a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | perl -pe 's%^([/a-zA-Z_-]+):.*?(##)%$$1 $$2%' | awk -F " *?## *?" '{printf "\033[36m%-30s\033[0m %-50s %s\n", $$1, $$2, $$3}'

