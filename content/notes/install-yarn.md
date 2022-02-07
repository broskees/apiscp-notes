---
title: Install Yarn
tags:
  - yarn
  - nodejs
emoji: 🐱
link: https://github.com/thundersquared/apiscp-yarn
queries:
  - how to install yarn
---

You can install Yarn via addin.

```bash
cd /usr/local/apnscp/resources/playbooks
git clone https://github.com/thundersquared/apiscp-yarn.git addins/apiscp-yarn
ansible-playbook addin.yml --extra-vars=addin=apiscp-yarn
```
