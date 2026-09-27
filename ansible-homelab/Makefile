# Makefile for ansible-homelab (powered by uv)

UV ?= uv
VAULT_PASS_FILE ?= ~/.vault_pass
PLAYBOOK = site.yml
INVENTORY = inventory/hosts.yml
LIMIT ?=

LIMIT_ARG = $(if $(LIMIT),--limit $(LIMIT),)

.PHONY: help install ping dry-run deploy check lint

help:
	@echo "Ansible Homelab Management Commands (via uv):"
	@echo "  make dry-run                   - Perform a dry run (--check --diff) of all hosts"
	@echo "  make dry-run LIMIT=desktop-tower - Perform a dry run targeting ONLY desktop-tower"
	@echo "  make deploy LIMIT=desktop-tower  - Run playbook targeting ONLY desktop-tower"
	@echo "  make ping                      - Ping all hosts (or specify LIMIT=desktop-tower)"
	@echo "  make check                     - Run dry-run (alias)"
	@echo "  make install                   - Install Ansible Galaxy collection requirements"
	@echo "  make lint                      - Run ansible-lint if installed"

install:
	$(UV) run ansible-galaxy collection install -r requirements.yml

ping:
	$(UV) run ansible $(if $(LIMIT),$(LIMIT),all) -i $(INVENTORY) -m ping --vault-password-file $(VAULT_PASS_FILE)

dry-run:
	$(UV) run ansible-playbook -i $(INVENTORY) $(PLAYBOOK) --vault-password-file $(VAULT_PASS_FILE) --check --diff $(LIMIT_ARG)

check: dry-run

deploy:
	$(UV) run ansible-playbook -i $(INVENTORY) $(PLAYBOOK) --vault-password-file $(VAULT_PASS_FILE) $(LIMIT_ARG)

lint:
	$(UV) run ansible-lint $(PLAYBOOK)
