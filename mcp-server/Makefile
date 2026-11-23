# Copyright (c) 2026, NVIDIA CORPORATION.  All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

IS_GO_MODULE := 1

include ../make/common.mk
include ../make/go.mk

.PHONY: all
all: lint-test

.PHONY: docs
docs: ## Regenerate docs/tools.md and docs/tools.json from the server
	$(GO) test -count=1 ./pkg/server -run TestToolDocsMatchTheServer -update

.PHONY: e2e
e2e: ## Run the scenarios through the binary on a kind cluster (see README)
	$(GO) build -o bin/mcp-server .
	E2E_BINARY=$(CURDIR)/bin/mcp-server $(GO) test -tags e2e -count=1 -v -timeout 40m ./tests/e2e/

.PHONY: help
help:
	@echo "mcp-server Makefile - Using nvsentinel make/*.mk standards"
	@echo ""
	@echo "Main targets: all, lint-test, build, test, lint, docs, e2e, clean"
	@echo "Ko targets: ko-build, ko-publish"
