SHELL := bash
.SHELLFLAGS := -eu -o pipefail -c
.DELETE_ON_ERROR:
MAKEFLAGS += --warn-undefined-variables
MAKEFLAGS += --no-builtin-rules
.SUFFIXES:

BLACKARGS := -t py36 aqt tests setup.py --exclude='aqt/forms|buildinfo'
ISORTARGS := aqt tests setup.py

$(shell mkdir -p .build ../dist)

PHONY: all
all: check

.build/run-deps: setup.py
	pip install -e .
	@touch $@

.build/dev-deps: requirements.dev
	pip install -r requirements.dev
	@touch $@

.build/ui: $(shell find designer -type f)
	./tools/build_ui.sh
	@touch $@

.build/i18n: $(wildcard i18n/translations/anki.pot/*)
	(cd i18n && ./build-mo-files && ./copy-qt-files)
	@touch $@

TSDEPS := $(wildcard ts/src/*.ts)

.build/js: $(TSDEPS)
	(cd ts && npm i && npm run build)
	@touch $@

BUILD_STEPS := .build/run-deps .build/dev-deps .build/js .build/ui .build/i18n aqt/buildinfo.py

# Checking
######################

.PHONY: check
check: $(BUILD_STEPS) .build/mypy .build/test .build/fmt .build/imports .build/lint .build/ts-fmt

.PHONY: fix
fix: $(BUILD_STEPS)
	isort $(ISORTARGS)
	black $(BLACKARGS)
	(cd ts && npm run pretty)

.PHONY: clean
clean:
	rm -rf .build aqt.egg-info build dist

# Checking Typescript
######################

JSDEPS := $(patsubst ts/src/%.ts, web/%.js, $(TSDEPS))

.build/ts-fmt: $(TSDEPS)
	(cd ts && npm i && npm run check-pretty)
	@touch $@

# Checking python
######################

PYLIB := ../pylib

CHECKDEPS := $(shell find aqt tests -name '*.py' | grep -v buildinfo.py)

.build/mypy: $(CHECKDEPS) .build/qt-stubs
	MYPYPATH=$(PYLIB) mypy aqt
	@touch $@

.build/test: $(CHECKDEPS)
	python -m pytest -s
	@touch $@

.build/lint: $(CHECKDEPS)
	pylint -j 0 --rcfile=.pylintrc -f colorized --extension-pkg-whitelist=PyQt5,ankirspy aqt tests setup.py
	@touch $@

.build/imports: $(CHECKDEPS)
	isort $(ISORTARGS) --check
	@touch $@

.build/fmt: $(CHECKDEPS)
	black --check $(BLACKARGS)
	@touch $@

.build/qt-stubs:
	./tools/typecheck-setup.sh
	@touch $@

# Building
######################

# we only want the wheel when building, but passing -f wheel to poetry
# breaks the inclusion of files listed in pyproject.toml
.PHONY: build
build: $(BUILD_STEPS)
	rm -rf dist
	python setup.py -q bdist_wheel
	rsync -a dist/*.whl ../dist/

.PHONY: develop
develop: $(BUILD_STEPS)

aqt/buildinfo.py: ../meta/version ../meta/buildhash
	echo "buildhash='$$(cat ../meta/buildhash)'" > $@
	echo "version='$$(cat ../meta/version)'" >> $@
