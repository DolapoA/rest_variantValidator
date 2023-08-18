# Running rest_VariantValidator in Docker

## Prerequisites

To install rest_variantValidator via Docker, first ensure that you have both docker and docker-compose installed. 
See their [documentation](https://docs.docker.com/compose/install/) for information.


## Clone the rest_VariantValidator Repository
Create a directory to collate your cloned repositories. Move into the directory, then clone the repository. 

```bash
$ git clone https://github.com/openvar/rest_variantValidator
```

Once the repository has been cloned, cd into the rest_variantValidator directory that the clone creates.

```bash
$ cd rest_variantValidator
``` 

If you have cloned the repository previously, update it prior to installing/re-installing using Docker

```bash
$ git pull
```

## Configuring the software

Edit the file located in `configuration/docker.ini`
You will need to provide an email address

**Optional (from VariantValidator v2.0.0 - September 2021)** Generate an Entrez API key. This will be necessary if you 
do not update your container for more than 12 months; else leave as `None`. See 
[Entrez API key](https://ncbiinsights.ncbi.nlm.nih.gov/2017/11/02/new-api-keys-for-the-e-utilities/) for details

Note: Reconfiguration can be achieved by accessing the docker container through bash. See below for entry and the 
VariantValidator [manual](https://github.com/openvar/variantValidator/blob/master/docs/MANUAL.md) for details

## Build the container

*Note: some of these steps take >>1hr to complete depending on the speed of your internet connection, particularly 
compiling SeqRepo*

- Pull images

```bash
$ docker-compose pull
```

- Create a directory for sharing resources between your computer and the container
```bash
$ mkdir ~/variantvalidator_data
$ mkdir ~/variantvalidator_data/share
$ mkdir ~/variantvalidator_data/share/seqrepo/
$ mkdir ~/variantvalidator_data/share/logs
```
*i.e.* a directory called `variantvalidator_data/share` in your `home` directory

- Build

```bash
$ docker-compose build --no-cache
```
 
- Complete build
    - The first time you do this, it will complete the build process, for example, populating the required the databases
    - The build takes a while because the  vv databases are large. However, this is a significant improvement on previou
    s versions. Build time is ~30 minutes (depending on the speed of you computer and internet connection)
    - The build has completed when you see the message ***"Use 'docker scan' to run Snyk tests against images to find vulnerabilities and learn how to fix them"***

- Test each container and completes builds if necessary
```bash
# If you have previously installed this software you will need to remove old SeqRepo databases
$ rm -r -f ~/variantvalidator_data/share/seqrepo/<Previos_SeqRepo_Directory>

# Create the vvta container (This takes ~10 minutes to complete)
$ docker-compose up vvta

# When you see the following message the container has been created (Can take ~30 min or so). 
"database system is ready to accept connections"

# Then perforn shut down 
ctrl + c

# Create the vdb container (This takes a few minutes)
$ docker-compose up vdb

# When you see the following message the container has been created. 
" /usr/sbin/mysqld: ready for connections"

# Then perform shut down 
ctrl + c

# Create the SeqRepo comtainer
docker-compose up seqrepo

# This will auto exit once complete and you will see
"exited with code 0"

# Start the container
$ docker-compose up -d

# Start the server
$ docker exec -it rest_variantvalidator-restvv-1 gunicorn  -b 0.0.0.0:8000 --timeout 600 app --threads=5 --chdir ./rest_VariantValidator/

# Now go to Running the app (below) if you have no build errors
```

### Build errors you may encounter

***If you have MySQL and or Postgres databases already running, you may encounter an error***  

> "ERROR: for vdb  Cannot start service vdb: Ports are not available: listen tcp 0.0.0.0:3306: bind: address already in use" 

If you encounter these issues, stop the build by pressing `ctrl+c`

- Reconfigure the ports used in the `docker-comose.yml` file as shown here
```yml
services:
  vdb:
    build:
      context: .
      dockerfile: vdb_docker.df
    ports:
      # - "33060:3306"
      - "3306:3306"
    expose:
      # - "33060"
      - "3306"
  uta:
    build:
      context: .
      dockerfile: uta_docker.df
    ports:
      - "54320:5432"
    expose:
      - "54320"

``` 
- hash (`#`) the conflicting port and add the new ports as shown above
- force-recreate the container

```bash
$ docker-compose down
$ docker-compose up --force-recreate
```

***You may encounter a build error relating to other unavailable ports***  

> "Cannot start service restvv: Ports are not available: listen tcp 0.0.0.0:8000: bind: address already in use" 

If you encounter these issues, stop the build by pressing `ctrl+c`

- Reconfigure the ports used in the `docker-comose.yml` file as shown here

```yml
  restvv:
    build: .
    depends_on:
      - vdb
      - uta
    volumes:
      - seqdata:/usr/local/share/seqrepo
    ports:
      - "5000:5000"
      # - "8000:8000"
      - "8080:8080"
    expose:
      - "5000"
      # - "8000"
      - 8080
```

- hash (`#`) the conflicting port and add the new ports as shown above
- Change the command in Dockerfile to reflect the changes e.g. `CMD gunicorn  -b 0.0.0.0:8080 app --threads=5 --chdir ./rest_VariantValidator/`
- force-recreate the container

```bash
$ docker-compose down
$ docker-compose up --force-recreate
```

## Running the app

```bash
# Start the container
$ docker-compose up -d

# Start the server
$ docker exec -it rest_variantvalidator-restvv-1 gunicorn  -b 0.0.0.0:8000 --timeout 600 app --threads=5 --chdir ./rest_VariantValidator/
```

In a web browser navigate to
[http://0.0.0.0:8000](http://0.0.0.0:8000)

When you are finished, stop the container
```
ctrl + c
```

***Note: you may need to change :8080 to one of :5000 or :8000 depending on whether you altered the default port above***


## Accessing the VariantValidator databases externally
It is possible to access both the UTA and Validator databases outside of docker as they expose the
 default PostgreSQL and MySQL ports (5432 and 3306 respectively). In the current set-up it is not possible to 
 access the seqrepo database outside of docker.
 

## Accessing VariantValidator directly through bash and reconfiguring a container post build
The container hosts a full install of VariantValidator. 

To start this version you start the container in detached mode and access it using

```bash
$ docker-compose exec restvv bash
```

When you are finished exit the container

```bash
$ exit
```

#### What you can do in bash mode

1. Run VariantValidator can be run on the commandline from within the container
    - Instructions can be found in the VariantValidator [manual](https://github.com/openvar/variantValidator/blob/master/docs/MANUAL.md) under sections **Database updates** and **Operation**
    
2. Start the REST services in development mode, bound to port 5000 
    - For example, this is useful if you want to develop new methods and test them
    - Note: Under the terms and conditions of our [license](https://github.com/openvar/rest_variantValidator/blob/master/LICENSE.txt) changes to the code and improvements must be made available to the community so that we can integrate them for the good of all our users 
    - See instructions on VariantValidator development in Docker 

3. For other commands that you can run using bash, see the [VariantValidator DOCKER.md](https://github.com/openvar/variantValidator/blob/master/docs/DOCKER.md)

## Developing VariantValidator in Docker
The container has been configured with git installed. This means that you can clone Repos directly into the container

To develop VariantValidator in the container

Start the container in detached mode

```bash
$ docker-compose exec restvv bash
```

ON YOUR COMPUTER change into the share directory

```bash
$ cd ~/share
```

Then create a directory for development

```bash
$ mkdir DevelopmentRepos
$ cd ~/share/DevelopmentRepos
```

Clone the VariantValidator Repo

```bash
$ git clone https://github.com/openvar/variantValidator.git
```

Checkout the develop branch

```bash
$ git checkout develop
$ git pull
```

Create an new branch for your developments

```bash
$ git branch name_of_branch
$ git checkout name_of_branch
```

IN THE CONTAINER, pip install the code so it can be run by the container

```bash
$ cd /usr/local/share/DevelopmentRepos/variantValidator
$ pip install -e . 
```

You can then use the containers Python interpreter to run queries, e.g.

```python
import json
import VariantValidator
vval = VariantValidator.Validator()
variant = 'NM_000088.3:c.589G>T'
genome_build = 'GRCh38'
select_transcripts = 'all'
validate = vval.validate(variant, genome_build, select_transcripts)
validation = validate.format_as_dict(with_meta=True)
print(json.dumps(validation, sort_keys=True, indent=4, separators=(',', ': ')))
```

## Developing rest_VariantValidator in Docker
The process for cloning the repo is the same as for VariantValidator

```bash
$ cd ~/share/DevelopmentRepos
$ git clone https://github.com/openvar/rest_variantValidator.git
```

Also, branches are created in the same way 

```bash
$ git checkout develop
$ git pull
$ git branch name_of_branch
$ git checkout name_of_branch
```

Navigating to the Repo is identical

```bash
$ docker-compose exec restvv bash
$ cd /usr/local/share/DevelopmentRepos/rest_variantValidator
```

However, instead of running `pip install -e`, we can test the install using the Python development server

```bash
python rest_variantValidator/app.py
```

## Updating rest_variantValidator
To update a container, use

```bash
$ docker-compose down
$ docker-compose build <service name> --no-cache
$ docker-compose up <service name>
```

## Deleting rest_variantValidator
Update requires that the restvv container is deleted from your system. This is not achieved by removing the container

If you are only running rest_variantValidator in docker, we recommend deleting and re-building all containers

```bash
# Remove the specific containers
$ docker-compose rm <service name e.g. vvta> 

# OR Delete all containers on your system
$ docker-compose down
$ docker system prune -a --volumes
```

***If you choose this option, make sure you see the container restvv being re-created and all Python packages being 
reinstalled in the printed logs, otherwise the container may not actually be rebuilt and the contained modules may not
 update***
