# ******************************************************************************
# Copyright 2017-2018 Intel Corporation
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
# ******************************************************************************

# Basic Makefile for contrib/docker. This can be expanded later as more targets
# are added.

# Building LLVM from source has been observed to trigger the oom-killer
#    on systems with a large number of cores
#    running with make -j
#
# Default is to build with -j 22 for parallel cmake/make.
# Override with make PARALLEL="-j <num_parallel_processes>" where
#    <num_parallel_processes> = the number of make processes to run in parallel
# Turn off with make PARALLEL=
PARALLEL=-j 22 

# DIR is an internal variable that serves as an anchor to this cloned git
# repository.  DIR is mounted into the docker container, so that builds
# can occur within the container on this cloned git repository.  DIR should
# not be modified - if it is, then the build system will not work.
DIR = $(realpath ../..)

# DOCKUSER_HOME is the location of the home directory of the fabricated
# "dockuser" user, used only within the docker containers.  "dockuser" is
# created (from the passed-in RUN_UID) to map the docker-caller user's UID to a
# first-class user (/etc/passwd entry, member of sudo group, proper home dir)
# /home/dockuser is also used in other scripts, notably run_as_user.sh, so if
# changed it must be done in other areas for the builds to work.
DOCKUSER_HOME=/home/dockuser

# Use /home/dockuser/ngraph-test, because we run as the user (and not root)
# /root/ngraph-test is not used, because /root is not accessible to user
VOLUME = -v "${DIR}:${DOCKUSER_HOME}/ngraph-test"
GIT_COMMIT = $(shell git rev-parse HEAD)
DBUILD_VERSION = ${GIT_COMMIT}_${PYTHON_VERSION}
DBUILD_DIR = ${DIR}/contrib/docker/.build-${DBUILD_VERSION}

# Enable additional options to be added on the command line
ifndef CMAKE_OPTIONS_EXTRA
    CMAKE_OPTIONS_EXTRA=
endif

# OS set to 'ubuntu1604' by default
# can be overridden on the command line with 'make <target> OS=centos74"
ifndef OS
    OS="ubuntu1604"
endif

ifeq ("$(shell echo ${OS} | grep centos)","centos74")
    RUN_AS_USER_SCRIPT=${DOCKUSER_HOME}/ngraph-test/contrib/docker/run_as_centos_user.sh
    DOCKERFILE=Dockerfile.ngraph.centos74_cmake3
else
    DOCKERFILE ?= "Dockerfile.ngraph"
    RUN_AS_USER_SCRIPT ?= ${DOCKUSER_HOME}/ngraph-test/contrib/docker/run_as_ubuntu_user.sh
endif

# For gcc builds, we do NOT regard warnings as errors
# For clang builds, we DO make warnings into errors
CMAKE_OPTIONS_COMMON=-DNGRAPH_BUILD_DOXYGEN_DOCS=ON -DNGRAPH_BUILD_SPHINX_DOCS=ON -DCMAKE_BUILD_TYPE=RelWithDebInfo $(CMAKE_OPTIONS_EXTRA)
CMAKE_OPTIONS_GCC=$(CMAKE_OPTIONS_COMMON) -DNGRAPH_INSTALL_PREFIX=${DOCKUSER_HOME}/ngraph-test/BUILD-GCC/ngraph_dist
CMAKE_OPTIONS_CLANG=$(MAKE_OPTIONS_COMMON)-DNGRAPH_INSTALL_PREFIX=${DOCKUSER_HOME}/ngraph-test/BUILD-CLANG/ngraph_dist -DCMAKE_CXX_COMPILER=clang++-3.9 -DCMAKE_C_COMPILER=clang-3.9 -DNGRAPH_WARNINGS_AS_ERRORS=ON -DNGRAPH_USE_PREBUILT_LLVM=TRUE

CALLER_UID := $(shell id -u)
CALLER_GID := $(shell id -g)

# Default version is python 2, but can be switched to 3 from command
# line
PYTHON_VERSION = 2

# Some targets are DEPRECATED and will be removed at a later date: check_cpu build_ngraph_cpu
# These DEPRECATED targets are currently included for Jenkins job compatibility with older dev branches
# Please see comments for individual targets for more details
.PHONY: clean build_docker_image build_gcc check_gcc build_clang check_clang install_gcc install_clang shell check_cpu build_all build_ngraph_cpu

DOCKER_BUILD=docker build --rm=true

ifdef http_proxy
DOCKER_BUILD+=--build-arg http_proxy=$(http_proxy)
DOCKER_RUN_ENV+=--env "http_proxy=$(http_proxy)"
endif

ifdef https_proxy
DOCKER_BUILD+=--build-arg https_proxy=$(https_proxy)
DOCKER_RUN_ENV+=--env "https_proxy=$(https_proxy)"
endif

all: check_gcc check_clang

# Docker actions

expand_dockerfile_templates:
	@echo "OS=${OS}"
	@echo "DOCKERFILE=${DOCKERFILE}"
	@echo "RUN_AS_USER_SCRIPT=${RUN_AS_USER_SCRIPT}"
	cd "${DIR}"/contrib/docker
	mkdir "${DBUILD_DIR}" || true
	sed -e 's/\(FROM ngraph.*\)/\1:${DBUILD_VERSION}/' ${DOCKERFILE} > "${DBUILD_DIR}"/Dockerfile.build_ngraph

build_docker_image: expand_dockerfile_templates
	$(DOCKER_BUILD) -f="${DBUILD_DIR}"/Dockerfile.build_ngraph --build-arg python_version="${PYTHON_VERSION}" -t=build_ngraph:"${DBUILD_VERSION}" .
	# remove the tag for the previous latest image
	docker rmi build_ngraph:latest || echo "keep going if docker rmi command fails"
	docker tag `docker images -q "build_ngraph:${DBUILD_VERSION}"` build_ngraph:latest

build_docker: build_docker_image

# Build docs
docs: sphinx_doc

sphinx_doc: build_docker_image
	# sphinx html docs build
	docker run --rm --tty \
            ${VOLUME} \
	    {DOCKER_RUN_ENV} \
            --env RUN_UID="$(shell id -u)" \
            --env RUN_CMD="set -e ; set -o pipefail ; cd ${DOCKUSER_HOME}/ngraph-test/doc/sphinx; env VERBOSE=1 make html 2>&1 | tee make_sphinx_html.log" \
            "build_ngraph:${DBUILD_VERSION}" \
	    sh -c "cd ${DOCKUSER_HOME}; ${RUN_AS_USER_SCRIPT}"

# Build
build_all: build_gcc build_clang

build_gcc: build_docker_image
	# Remove old distribution directory if present
	( test -d "${DIR}"/BUILD-GCC/ngraph_dist && rm -fr "${DIR}"/BUILD-GCC/ngraph_dist && echo "Removed old ${DIR}/BUILD-GCC/ngraph_dist directory" ) || echo "Previous ngraph_dist directory not found"
	# Make BUILD-GCC directory as user
	mkdir -p "${DIR}"/BUILD-GCC
	chmod ug+rwx "${DIR}"/BUILD-GCC
	docker run --rm --tty \
            ${VOLUME} \
	    ${DOCKER_RUN_ENV} \
            --env GTEST_OUTPUT="xml:${DOCKUSER_HOME}/ngraph-test/BUILD-GCC/unit-test-results.xml" \
            --env RUN_UID="$(shell id -u)" \
            --env RUN_CMD="set -x; set -e ; set -o pipefail ; if [ -f "/etc/centos-release" ]; then cat /etc/centos-release; fi; if [ -f "/etc/lsb-release" ]; then cat /etc/lsb-release; fi; uname -a ; cat /etc/os-release || true; cd ${DOCKUSER_HOME}/ngraph-test/BUILD-GCC; cmake ${CMAKE_OPTIONS_GCC} .. 2>&1 | tee cmake_gcc.log ; env VERBOSE=1 make ${PARALLEL} 2>&1 | tee make_gcc.log" \
            "build_ngraph:${DBUILD_VERSION}" \
	    sh -c "cd ${DOCKUSER_HOME}; ${RUN_AS_USER_SCRIPT}"

build_clang: build_docker_image
	# Remove old distribution directory if present
	( test -d "${DIR}"/BUILD-CLANG/ngraph_dist && rm -fr "${DIR}"/BUILD-CLANG/ngraph_dist && echo "Removed old ${DIR}/BUILD-CLANG/ngraph_dist directory" ) || echo "Previous ngraph_dist directory not found"
	# Make BUILD-CLANG directory as user
	mkdir -p "${DIR}"/BUILD-CLANG
	chmod ug+rwx "${DIR}"/BUILD-CLANG
	docker run --rm --tty \
            ${VOLUME} \
	    ${DOCKER_RUN_ENV} \
            --env GTEST_OUTPUT="xml:${DOCKUSER_HOME}/ngraph-test/BUILD-CLANG/unit-test-results.xml" \
            --env RUN_UID="$(shell id -u)" \
            --env RUN_CMD="set -e ; set -o pipefail ; if [ -f "/etc/centos-release" ]; then cat /etc/centos-release; fi; if [ -f "/etc/lsb-release" ]; then cat /etc/lsb-release; fi; uname -a ; cat /etc/os-release || true; cd ${DOCKUSER_HOME}/ngraph-test/BUILD-CLANG; cmake ${CMAKE_OPTIONS_CLANG} .. 2>&1 | tee cmake_clang.log ; env VERBOSE=1 make ${PARALLEL} 2>&1 | tee make_clang.log" \
            "build_ngraph:${DBUILD_VERSION}" \
	    sh -c "cd ${DOCKUSER_HOME}; ${RUN_AS_USER_SCRIPT}"

# Check (run unit-tests)

check_all: check_gcc check_clang

check_gcc: build_gcc
	docker run --rm --tty \
            ${VOLUME} \
	    ${DOCKER_RUN_ENV} \
            --env GTEST_OUTPUT="xml:${DOCKUSER_HOME}/ngraph-test/BUILD-GCC/unit-test-results.xml" \
            --env RUN_UID="$(shell id -u)" \
            --env RUN_CMD="set -e ; set -o pipefail ; cd ${DOCKUSER_HOME}/ngraph-test/BUILD-GCC; env VERBOSE=1 make unit-test-check 2>&1 | tee make_check_gcc.log ; sed -E -e 's/classname\=\"[a-zA-Z0-9_]+/&1_gcc/' unit-test-results.xml > unit-test-results-gcc.xml" \
            "build_ngraph:${DBUILD_VERSION}" \
	    sh -c "cd ${DOCKUSER_HOME}; ${RUN_AS_USER_SCRIPT}"

check_clang: build_clang
	docker run --rm --tty \
            ${VOLUME} \
	    ${DOCKER_RUN_ENV} \
            --env GTEST_OUTPUT="xml:${DOCKUSER_HOME}/ngraph-test/BUILD-CLANG/unit-test-results.xml" \
            --env RUN_UID="$(shell id -u)" \
            --env RUN_CMD="set -e ; set -o pipefail ; cd ${DOCKUSER_HOME}/ngraph-test/BUILD-CLANG; env VERBOSE=1 make check 2>&1 | tee make_check_clang.log ; sed -E -e 's/classname\=\"[a-zA-Z0-9_]+/&1_clang/' unit-test-results.xml > unit-test-results-clang.xml" \
            "build_ngraph:${DBUILD_VERSION}" \
	    sh -c "cd ${DOCKUSER_HOME}; ${RUN_AS_USER_SCRIPT}"

style_clang: build_clang
	docker run --rm --tty \
            ${VOLUME} \
	    ${DOCKER_RUN_ENV} \
            --env GTEST_OUTPUT="xml:${DOCKUSER_HOME}/ngraph-test/BUILD-CLANG/unit-test-results.xml" \
            --env RUN_UID="$(shell id -u)" \
            --env RUN_CMD="set -e ; set -o pipefail ; cd ${DOCKUSER_HOME}/ngraph-test/BUILD-CLANG; env VERBOSE=1 make style-check 2>&1 | tee make_style_check_clang.log" \
            "build_ngraph:${DBUILD_VERSION}" \
	    sh -c "cd ${DOCKUSER_HOME}; ${RUN_AS_USER_SCRIPT}"

# Install

install_all: install_gcc install_clang

install_gcc: check_gcc
	# Puts ngraph_dist in BUILD-GCC directory.  This is used by Jenkins ngraph-tensorflow batch job.
	docker run --rm --tty \
            ${VOLUME} \
	    ${DOCKER_RUN_ENV} \
            --env RUN_UID="$(shell id -u)" \
            --env RUN_CMD="set -e ; set -o pipefail; cd ${DOCKUSER_HOME}/ngraph-test/BUILD-GCC ; test -d ngraph_dist && rm -fr ngraph_dist && echo 'Removed old ngraph_dist directory' ; make install 2>&1 | tee make_install_gcc.log ; tar czf ngraph_dist_gcc.tgz ngraph_dist 2>&1 | tee make_tarball_gcc.log" \
            "build_ngraph:${DBUILD_VERSION}" \
	    sh -c "cd ${DOCKUSER_HOME}; ${RUN_AS_USER_SCRIPT}"

install_clang: check_clang
	# Puts ngraph_dist in BUILD-CLANG directory.  This is used by Jenkins ngraph-tensorflow batch job.
	docker run --rm --tty \
            ${VOLUME} \
	    ${DOCKER_RUN_ENV} \
            --env RUN_UID="$(shell id -u)" \
            --env RUN_CMD="set -e ; set -o pipefail; cd ${DOCKUSER_HOME}/ngraph-test/BUILD-CLANG ; test -d ngraph_dist && rm -fr ngraph_dist && echo 'Removed old ngraph_dist directory' ; make install 2>&1 | tee make_install_clang.log ; tar czf ngraph_dist_clang.tgz ngraph_dist 2>&1 | tee make_tarball_clang.log" \
            "build_ngraph:${DBUILD_VERSION}" \
	    sh -c "cd ${DOCKUSER_HOME}; ${RUN_AS_USER_SCRIPT}"

# Interactive shell

shell: build_docker_image
	# "make shell" runs an interactive shell in the docker image, for debugging
	docker run --rm --tty --interactive \
            ${VOLUME} \
	    ${DOCKER_RUN_ENV} \
            --env RUN_UID="$(shell id -u)" \
            "build_ngraph:${DBUILD_VERSION}" \
	    sh -c "cd ${DOCKUSER_HOME}; ${RUN_AS_USER_SCRIPT}"

# Clean

clean:
	rm -f "${DIR}"/contrib/docker/.build-*/Dockerfile.* || echo "keep going if files are not present"
	rmdir "${DIR}"/contrib/docker/.build-* || echo "keep going if directory is not present"
	rm -fr "${DIR}"/BUILD-GCC
	rm -fr "${DIR}"/BUILD-CLANG

#
# DEPRECATED TARGETS -- These WILL BE REMOVED in a future revision.
#                       They exist here to maintain compatibility in Jenkins
#                       jobs on older development branches.
#

check_cpu: check_all
	echo 'WARNING: "make check_cpu" is DEPRECATED and will be removed in a future revision'
	echo '         "make check_cpu" runs "make check_all" now, building with all compilers (gcc and clang)'

build_ngraph_cpu: build_docker_image
	echo 'WARNING: "make build_ngraph_cpu" is DEPRECATED and will be removed in a future revision'
	echo '         "make build_ngraph_cpu" runs "make build_docker_image" now'
