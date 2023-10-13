# Introduction

Purpose of this project is to provide software for storing and using keys both physically and digitally secure way with IBM 4769 HSM. This software is released as Docker image containing only the application binary and needed shared libraries for it. This application is meant to be run in RHEL 8 Linux Server which has IBM 4769 HSM installed.

From these source codes two versions of the secops service applications are build and released.

1. Product SecOps Service application, this version provides interfaces for normal daily usage which includes data signing, encryption and decryption services.
2. Keyadmin SecOps Service application, this version provides interfaces for creating new product keys and modifying existing product keys

These applications are not run simultaneously and normally Product SecOps Service application is running. When there is a need for new product keys or need for modifications for existing product keys then Product SecOps Service application is stopped and Keyadmin SecOps Service is started for that time period. 


# Getting Started

### 1. Installation
Application is installed as a part of whole SecOps Server installation. Instructions for installation can be found from SecOps Server Installation Guide. Example configuration for cca_monitoring can be found from [SecOps Configuration Project](https://dev.azure.com/ABB-MO-Drives/FI%20GIT/_git/Product_SecOps_service_configurations). 

### 2. Software dependencies
- IBM 4769 drivers


# Build and Test
This project uses Azure DevOps Pipeline to build and test the application. Currently testing phase uses cppcheck and clang-tidy for static code analysis and unit tests are implemented with gtest framework. Code coverage is calculated with gcovr from unit test results and published in Azure DevOps Pipeline. All building and testing that is done in Azure DevOps Pipeline can also be done locally by developer and below are instructions how to do all that locally.

## Building locally

For building the applications locally you need to have Docker. Instructions for installing WSL and Docker can be found [here](https://odin-confluence.abb.com/x/E0GvHw/).

### 1. Get source code project
Clone the projects *Product_SecOps_service_SW* and *Product_SecOps_service_configurations* on the WSL Ubuntu host. If you want to use the create_container.sh [script](https://dev.azure.com/ABB-MO-Drives/FI%20GIT/_git/Product_SecOps_service_configurations?path=/imagebuild/dev_build/create_container.sh) from the configurations repository, you must use a directory, which is located in path ~/projects (create the projects-directory if it does not exist). For example:

    cd ~/projects
    git clone https://ABB-MO-Drives@dev.azure.com/ABB-MO-Drives/FI%20GIT/_git/Product_SecOps_service_SW
    git clone https://ABB-MO-Drives@dev.azure.com/ABB-MO-Drives/FI%20GIT/_git/Product_SecOps_service_configurations


### 2. Get Development docker image

The development docker image includes all needed tools to build and test the application. Development images are stored in [Azure Container Registy](https://portal.azure.com/#@ABB.onmicrosoft.com/resource/subscriptions/6b0a9ae0-e962-4ce1-b349-4a9c023f5af2/resourceGroups/SecOpsServiceAcr-rg/overview). Image can be pulled from the registry with docker pull if the user has needed access rights to the registry. 

Log into the registry (note that currently this does not work from ABB office network):
    
    az login --use-device-code
    az acr login -n secopsserviceacr

Use docker pull to get the image. Always use the latest image version in the registry. For example:
    
    docker pull secopsserviceacr.azurecr.io/secops_service/rhel_dev_build_image:2022.10.19.2

Next, developer can use the create_container.sh [script](https://dev.azure.com/ABB-MO-Drives/FI%20GIT/_git/Product_SecOps_service_configurations?path=/imagebuild/dev_build/create_container.sh) for creating suitable container from the pulled image.

    ./create_container.sh <container_name> <image_name>

Finally, start new development container.

    docker start -ia <container_name>




### 3. Build the project
Project has a script file called pipeline.sh in its root directory. That script can be used for building the software, e.g.

    ./pipeline.sh build release

Note that currently, if using from ABB office network, you must define a proxy server for the first build:

    export https_proxy=http://proxy-swg-geolb.abb.com:8082
    or
    export https_proxy=http://finland125184-swg.ibosscloud.com:8082

There are multiple versions that can be built for both Product SecOps Service and Keyadmin SecOps Service.

 | Product SecOps Service | Keyadmin Secops Service | Purpose
 |------|----|----|
 |release|keyadmin-release|Release version
 |debug|keyadmin-debug|Debug version
 |test|keyadmin-test|Unit test version
 |no-hsm|keyadmin-no-hsm|Testing without real HSM



## Testing locally
Same development docker container must be used for testing. Testing is divided into three different steps. All steps can be run with same pipeline.sh script as what is used for building the application.

### 1. Unit tests
Once test build has been done, unit tests can be run with command:

    ./pipeline.sh test unit
    or
    ./pipeline.sh test keyadmin-unit

Those commands run all unit test cases for selected build.
Once tests for both builds has been run, coverage report can be generated with command:

    ./pipeline test summary

Report is generated to the build-coverage-summary directory. From there user can find the summary file called index.html. That file can then be examined in browser and there are also report files for each individual source code files that are included into the tests. From those files user can check the line and branch coverages.

### 2. Static code analysis with cppcheck
Cppcheck is used for static code analysis and it can be run with command:

    ./pipeline.sh test static

At this step it is checked that there are no uncommitted changes in workarea, source file formatting is checked with clang-format and finally cppcheck is run. Source file formatting rules are described in file .clang-format which is located in the project root. Cppcheck rules are described in cppcheck.cmake file which is also located in the project root.

### 3. Code analysis with clang-tidy
Clang-tidy is used for another static code analysis and it can be run with command:

    pipeline.sh test tidy

Configuration for clang-tidy is stored in file .clang-tidy which is located in the project root. 


## Creating docker scratch image
It is also possible to create docker scratch image with pipeline.sh script. Creating the image is divided into two parts because development container doesn't include docker functionality and docker images cannot be created in it.

### Part 1.
In this part application binary and needed shared libraries are copied to the correct directory structure and packaged as tar file for next part. This part must be done in development container to get the needed libraries included into the package. Tar package can be created with command:

    ./pipeline.sh prepare scratch release
    or
    ./pipeline.sh prepare scratch keyadmin-release

As a result of that we get file called prepared_files_<build-name>.tar.gz which will be used in next part for creating the docker image.

### Part 2.
In this part file from previous part will be extracted and docker image will be created. This part must be done in host as docker container doesn't include docker in it. Before this can be run directories ./podman_scratch/lib64 and ./podman_scracth/opt must be removed as the tar package will be extracted there (Note: sudo is needed for removing the /lib64 -directory). Docker image can be created and saved as a local tar package with command:

    ./pipeline.sh build scratch release save
    or
    ./pipeline.sh build scratch keyadmin-release save

As a result of that we get file product_secops_service-release-yyyy-mm-dd-hhmmss.tar or keyadmin_secops_service-release-yyyy-mm-dd-hhmmss.tar accordingly.
