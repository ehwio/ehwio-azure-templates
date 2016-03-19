all:
	../../utils/compile.py azure.deployment.yaml
deploy: all
	azure group deployment create -f azure.deployment.json \
	                              -e azure.deployment.parameters.json \
																${RG}
