# This Makefile stub allows you to customize starter pack (SP) targets.
# Consider this file as a bridge between your project
# and the starter pack's predefined targets that reside in Makefile.sp.
#
# You can add your own, non-SP targets here or override SP targets
# to fit your project's needs. For example, you can define and use targets
# named "install" or "run", but continue to use SP targets like "sp-install"
# or "sp-run" when working on the documentation.

current_dir := $(dir $(abspath $(firstword $(MAKEFILE_LIST))))

# Set the path prefix and MicroCloud component versions corresponding to each MicroCloud docs version
PATH_PREFIX      = /en/latest/

MATTERVERSION	= origin/main
INDUSTRIAL_DOCS_VERSION = origin/main
EDGE_AI_VERSION = origin/main



# Put it first so that "make" without argument is like "make help".
help:
	@echo "\n" \
        "------------------------------------------------------------- \n" \
        "* watch, build and serve the documentation:  make run \n" \
        "* only build:                                make html \n" \
        "* only serve:                                make serve \n" \
        "* clean built doc files:                     make clean-doc \n" \
        "* clean full environment:                    make clean \n" \
        "* check links:                               make linkcheck \n" \
        "* check spelling:                            make spelling \n" \
        "* check spelling (without building again):   make spellcheck \n" \
        "* check inclusive language:                  make woke \n" \
        "* check accessibility:                       make pa11y \n" \
        "* check style guide compliance:              make vale \n" \
        "* check style guide compliance on target:    make vale TARGET=* \n" \
        "* other possible targets:                    make <TAB twice> \n" \
        "------------------------------------------------------------- \n"

integrate:
	# Pull the other repositories

	@echo "\033[0;34mStarting integration and cloning repositories...\033[0m"


	# cd integration/ && ( git -C matter-docs fetch || git clone https://github.com/canonical/matter-docs.git)
	# git -C integration/matter-docs reset --hard $(MATTERVERSION)


	# cd integration/ && ( git -C industrial_documentation fetch || git clone https://github.com/canonical/industrial_documentation.git )
	# git -C integration/industrial_documentation reset --hard $(INDUSTRIAL_DOCS_VERSION)

	# cd integration/ && ( git -C edge_ai_docs fetch || git clone https://github.com/geoffreynyaga/edge_ai_docs.git )
	# git -C integration/edge_ai_docs reset --hard $(EDGE_AI_VERSION)


	# Create a directory for files to override in MicroCloud docs
	mkdir -p integration/microcloud/_templates/ integration/microcloud/_static
	
	# Copy the header HTML and CSS files for the doc sets
	cp .sphinx/_integration/microcloud.html integration/microcloud/_templates/header.html
	cp .sphinx/_integration/header.css integration/microcloud/_static/

	cp .sphinx/_integration/matter.html integration/matter-docs/.sphinx/_templates/header.html
	cp .sphinx/_integration/header.css integration/matter-docs/.sphinx/_static/

	cp .sphinx/_integration/industrial.html integration/industrial_documentation/.sphinx/_templates/header.html
	cp .sphinx/_integration/header.css integration/industrial_documentation/.sphinx/_static/

	cp .sphinx/_integration/edge-ai.html integration/edge_ai_docs/.sphinx/_templates/header.html
	cp .sphinx/_integration/header.css integration/edge_ai_docs/.sphinx/_static/

	cp .sphinx/_integration/smart_cities.html integration/smart_cities/.sphinx/_templates/header.html
	cp .sphinx/_integration/header.css integration/smart_cities/.sphinx/_static/



	# Copy the search JavaScript file
	cp .sphinx/_integration/rtd-search.js integration/microcloud/_static/

	cp .sphinx/_integration/rtd-search.js integration/matter-docs/.sphinx/_static/
	cp .sphinx/_integration/rtd-search.js integration/industrial_documentation/.sphinx/_static/
	cp .sphinx/_integration/rtd-search.js integration/edge_ai_docs/.sphinx/_static/
	cp .sphinx/_integration/rtd-search.js integration/smart_cities/.sphinx/_static/


	# Add information about where to find the tags/docs to the doc sets

	# cat .sphinx/_integration/add_config.py >> integration/matter-docs/custom_conf.py
	# cat .sphinx/_integration/add_config.py >> integration/industrial_documentation/custom_conf.py
	# cat .sphinx/_integration/add_config.py >> integration/edge_ai_docs/custom_conf.py
	# cat .sphinx/_integration/add_config.py >> integration/smart_cities/custom_conf.py

	

	@echo "\033[0;32mIntegration completed successfully!\033[0m"


clean-integrate:

	# rm -rf integration/matter-docs

clean: clean-integrate
	$(MAKE) -f Makefile.sp sp-clean

# `html` builds the integrated docs.
html: integrate
	mkdir -p $(current_dir)/_build

	cd integration/matter-docs/ && $(MAKE) html BUILDDIR=$(current_dir)/_build/matter-docs
	cd integration/edge_ai_docs/ && $(MAKE) html BUILDDIR=$(current_dir)/_build/edge_ai_docs
	cd integration/smart_cities/ && $(MAKE) html BUILDDIR=$(current_dir)/_build/smart_cities
	cd integration/industrial_documentation/ && $(MAKE) html BUILDDIR=$(current_dir)/_build/industrial_documentation
	
	$(MAKE) -f Makefile.sp sp-html BUILDDIR=$(current_dir)/_build/microcloud

# `html-rtd` builds the integrated docs, with the correct paths for Read the Docs.
# This target is used by the Read the Docs build.
html-rtd:

	PATH_PREFIX=$(PATH_PREFIX) $(MAKE) -C integration/matter-docs/ html BUILDDIR=$(READTHEDOCS_OUTPUT)/html/matter-docs
	PATH_PREFIX=$(PATH_PREFIX) $(MAKE) -C integration/industrial_documentation/ html BUILDDIR=$(READTHEDOCS_OUTPUT)/html/industrial_documentation
	PATH_PREFIX=$(PATH_PREFIX) $(MAKE) -C integration/edge_ai_docs/ html BUILDDIR=$(READTHEDOCS_OUTPUT)/html/edge_ai_docs
	PATH_PREFIX=$(PATH_PREFIX) $(MAKE) -C integration/smart_cities/ html BUILDDIR=$(READTHEDOCS_OUTPUT)/html/smart_cities

	PATH_PREFIX=$(PATH_PREFIX) $(MAKE) -f Makefile.sp sp-html BUILDDIR=$(READTHEDOCS_OUTPUT)/html/microcloud
# `spelling` checks only the MicroCloud docs.
spelling: clean-doc microcloud
	$(MAKE) -f Makefile.sp sp-spellcheck

# `microcloud` builds only the MicroCloud docs, as a stand-alone doc set.
microcloud:
	$(MAKE) -f Makefile.sp sp-html SINGLE_BUILD=True

.PHONY: run
run:
	$(MAKE) -f Makefile.sp sp-run LOCAL_SPHINX_BUILD=True ADDPREREQS='gitpython pyyaml' SPHINXOPTS="--ignore './index.html' -c . -d .sphinx/.doctrees -j auto"

# `serve` rebuilds the integrated docs and serves them.
serve: html
	cd $(current_dir)/_build/; python3 -m http.server --bind 127.0.0.1 8000

# `serve-microcloud` rebuilds the MicroCloud docs as a stand-along doc set and serves them.
serve-microcloud: microcloud
	cd $(current_dir)/_build/; python3 -m http.server --bind 127.0.0.1 8000

%:
	$(MAKE) -f Makefile.sp sp-$@
