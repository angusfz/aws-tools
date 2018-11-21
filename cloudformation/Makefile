.DEFAULT_GOAL := help
SHELL=/usr/bin/env bash -e -o pipefail

TMP_MSG := ".tmpmsg"
TMP_RET := ".tmpret"

CFN_TEMPLATE := "./cdpipeline.yaml"
CFN_STACKNAME := cdpipeline
CFN_PARAMETER_FILE := "./config.json"
CFN_TEMPLATEDIR = $(shell dirname $(CFN_TEMPLATE))

ifdef CFN_PARAMETER_FILE
CFN_PARAMETER-OVERRIDES := --parameter-overrides $(shell jq -j '.[] | "\"" + .ParameterKey + "=" + .ParameterValue +"\" "' $(CFN_PARAMETER_FILE))
endif

print-target = echo "$(shell tput bold;tput setaf 2 ) === $@ === $(shell tput sgr0)"
check-dependency = $(if $(shell command -v $(1)),,$(error Make sure $(1) is installed))

install: ## Install require tools and python lib ## -
	pip install awscli cfn-lint 
	sudo gem install cfn-nag

clean: ## Clean temporary file ## -
	@$(call print-target)
	rm -rf $(CFN_TEMPLATE_PACKAGED)
	rm -rf $(TMP_MSG)
	rm -rf $(TMP_RET)
	rm -rf $(CFN_TEMPLATE)

check-required-tools:
	@$(call print-target)
	@$(call check-dependency,aws)
	@$(call check-dependency,jq)
	@$(call check-dependency,cfn-lint)
	@$(call check-dependency,cfn_nag)
	@$(call check-dependency,tput)
	@echo "√ Pass"

run: ## Invoke run.py to generate cloudformation template ## -
	@$(call print-target)
	python run.py
	@make test

cfn-lint:check-required-tools
	@$(call print-target)
	@find $(CFN_TEMPLATEDIR) -type f \( -name "*.yaml" -or -name "*.yml" \) | \
		while read f; do ( cfn-lint -t $$f && echo "√ $$f") || exit 1 ; done

cfn-nag:cfn-lint
	@$(call print-target)
	@cfn_nag_scan --template-pattern="..*\.yaml|..*\.yml" --input-path=$(CFN_TEMPLATEDIR) --allow-suppression


test:cfn-nag

cfn-validate:test
	@$(call print-target)
	@find $(CFN_TEMPLATEDIR) -type f \( -name "*.yaml" -or -name "*.yml" \) | \
		while read f; do \
		( aws cloudformation validate-template --template-body file://$$f 1>/dev/null 2> $(TMP_MSG) && echo "√ $$f") || \
		( echo "✗ $$f" && cat $(TMP_MSG) && >$(TMP_MSG) && exit 1 ) ; done

createcs: ## Create and list cloudformation stack changeset ## -
	@$(call print-target)
	@$$((aws cloudformation deploy \
		--template-file $(CFN_TEMPLATE) \
		--stack-name $(CFN_STACKNAME) \
		--no-execute-changeset \
		$(CFN_PARAMETER-OVERRIDES) \
		--capabilities CAPABILITY_NAMED_IAM \
		1> $(TMP_MSG)  2>&1 ) ;echo $${PIPESTATUS[0]} > $(TMP_RET))
	@# Check return code and msg. If no changeset, cfn deploy will return 255 with msg  "No changes to deploy"
	@if [ $$(cat $(TMP_RET)) -eq 0 ]; then \
		cat $(TMP_MSG); \
		echo ""; \
		echo "=== List stack changeset for stack $(CFN_STACKNAME) ==="; \
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
deploy:createcs ## Deploy cloudformation stack ## -
	@$(call print-target)
	aws cloudformation deploy      \
		--template-file $(CFN_TEMPLATE) \
		--stack-name $(CFN_STACKNAME)	\
		--no-fail-on-empty-changeset	\
		$(CFN_PARAMETER-OVERRIDES)      \
		--capabilities CAPABILITY_NAMED_IAM \
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


