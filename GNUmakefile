TEST?=github.com/beamly/terraform-provider-gocd/gocd/
GOFMT_FILES?=$$(glide novendor)
SHELL:=/bin/bash

# For local testing, run `make testacc`
SERVER ?=http://127.0.0.1:8153/go/
TESTARGS ?= -race -coverprofile=profile.out -covermode=atomic

export GOCD_URL=$(SERVER)
export GOCD_SKIP_SSL_CHECK=1

## Travis targets
travis: before_install script after_success deploy_on_develop

before_install:
	go get -mod=readonly golang.org/x/lint/golint
	curl https://glide.sh/get | sh
	glide install

script: testacc

after_failure: cleanup

after_success: report_coverage cleanup
	-go get -mod=readonly github.com/goreleaser/goreleaser
	go get -mod=readonly github.com/goreleaser/goreleaser@v0.120.8

prepare_goreleaser:
	git clean -fd
	go get -mod=readonly
	git checkout -- go.mod go.sum

deploy_on_tag: prepare_goreleaser
	goreleaser --debug

deploy_on_develop: prepare_goreleaser
	goreleaser --debug --rm-dist --snapshot

## General Targets
teardown-test-gocd:
	rm -f godata/server/config/cruise-config.xml
	docker-compose down

cleanup: teardown-test-gocd 

report_coverage:
	bash <(curl -s https://codecov.io/bash)


default: build

build: fmtcheck
	go install -mod=readonly

test: fmtcheck
	TF_ACC=1 TESTARGS="$(TESTARGS)" bash ./scripts/go-test.sh

testacc: provision-test-gocd
	bash scripts/wait-for-test-server.sh
	TF_ACC=1 $(MAKE) test

vet:
	@echo "go vet ."
	@go vet $$(go list ./... | grep -v vendor/) ; if [ $$? -eq 1 ]; then \
		echo ""; \
		echo "Vet found suspicious constructs. Please check the reported constructs"; \
		echo "and fix them if necessary before submitting the code for review."; \
		exit 1; \
	fi

fmt:
	go fmt $(GOFMT_FILES)

fmtcheck:
	@sh -c "'$(CURDIR)/scripts/gofmtcheck.sh'"

errcheck:
	@sh -c "'$(CURDIR)/scripts/errcheck.sh'"

test-compile:
	@if [ "$(TEST)" = "./..." ]; then \
		echo "ERROR: Set TEST to a specific package. For example,"; \
		echo "  make test-compile TEST=./gocd"; \
		exit 1; \
	fi
	go test -c $(TEST) $(TESTARGS)

provision-test-gocd:
	cp godata/default.gocd.config.xml godata/server/config/cruise-config.xml
	docker-compose build --build-arg UID=$(shell id -u) gocd-server
	docker-compose up -d

.PHONY: build test testacc vet fmt fmtcheck errcheck vendor-status test-compile
