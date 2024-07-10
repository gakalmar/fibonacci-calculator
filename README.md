# fibonacci-calculator
- A simple fibonacci sequence calculator application created to showcase complex application deployment with automated CI/CD integration:
    - `React` frontend
    - `Express` backend
    - Store data in `redis` and in `postgres` (intentional complexity for practice purposes!)
    - Use `nginx` to route incoming requests in development

# start
- clone the repo:
    - `git clone https://github.com/gakalmar/fibonacci-calculator.git`
- start app simply with the following command in the project root:
    - `docker-compose up`

# development deployment guide
- Worker folder (calculate number based on index and connect to redis)
    - package.json
    - keys.js
    - index.js

- Server folder (connect to postgres and create table, connect to redis, express routes)
    - package.json
    - keys.js
    - index.js

- React app (`npx create-react-app client`)

- Create images for Development first (`Dockerfile.dev` in each folder):
    - Copy package.json
    - Run npm install
    - Copy the rest of the files
    - Set up volumes to share files with docker compose

- Connect these 3 services with `docker-compose`, and add other services:
    - Create **postgres** service (potential issue with image version!)
    - Create **redis** service (potential issue with image version!)
    - Create **server** service:
        - build from Dockerfile
        - add volumes:
            - tag `/app/node_modules` folder, so that is not overwritten!
            - reference `./server:/app`, so that the `app` folder in the `server` container is always what is in our machine's `server` folder
        - specify environmental variables:
            - `variableName=value` (sets variable at runtime) - USE THIS METHOD
            - `variableName` (in this case the value is taken from my local environment) - it's still set at runtime
            - potential issue - rename variables from `PGPASSWORD` to postgres documentation format `POSTGRES_PASSWPRD`
    - Create **client** service: Identical to server mostly
    - Create **worker** service: Identical to server mostly
    - Add **nginx**:
        - This is to create separate routing for backend and frontend requests
        - We do this by creating a `defult.conf` file in a separate nginx folder:
            - *upstream* servers will be on port 3000 and port 5000 (`client:3000` and `server:5000` -> these come from their corresponding index.js files!)
            - listen for incoming calls on port 80
            - `/` requests should be routed to the react upstream server
            - `/api` requests should be routed to the express upstream server (on the server the `/api` part is not there anymore, it's just used for the nginx routing!)
        - Add Dockerfile for nginx, to copy the config file accross!

# production deployment guide
- Create `Dockerfiles` for each service for **production** instead of the ones created for **development**
    - `worker`: only change CMD command to `npm run start` from `npm run dev`
    - `server`: only change CMD command to `npm run start` from `npm run dev`
    - `nginx`: no changesV
    - `client`:
        - architecutre that needs to be set up:
            - first a request gets to the `nginx` router, listening on port 80
            - this is responsible for forwarding the requests to the backends:
                - React build files for production, which are connected through another `nginx` image, listening on port 3000 
                    - this `nginx` will also need a `default.conf` file inside the client folder!
                - Express server, listening on port 5000

- Set up testing phase:
    - For now, we just clean the `App.test.js` file, but we would put the tests in here
    - The test is only set up to run tests for the `client`, but we could also add tests for `worker` and `server` here

- Connect github repo to Travis:
    - Go to `https://www.travis-ci.com/`, where if you have the connection already set to your github repo, you should see your project
    - Create the `.travis.yml` file, that includes the instructions for the pipeline:
        - Connect to docker
        - `before_install` section:
            - Create a test image for testing purposes (use the `dev` Dockerfile, because we also need the dependencies!)
        - `scripts`:
            - We do the testing in this phase. If a script returns with a result other than 0, the pipeline is stopped
        - `afrer_success`:
            - This section only runs if all tests were successful
            - We are building all 4 production images here (`client`, `server`, `worker`, `nginx`)
            - We are also pushing these images to `DockerHub`, for which we need to connect to the `Docker CLI`:
                - Docker login:
                    - to add the dockerID and password securely, we enter these as secrets in Travis:
                        - click on repo in the Travis UI, then on 'more options' -> 'settings' -> 'environment variables':
                            - Add name:
                                - name: `DOCKER_ID`
                                - value: `gakalmar`
                            - Add password:
                                - name: `DOCKER_PASSWORD`
                                - value: `<your password>`
                    - Now we can add the actual command:
                        - `echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_ID" --password-stdin`
                - Push the images

- Deployment to AWS Elastic Beanstalk:
    - `Create a Dockerrun.aws.json` file - this will be the instructions file for AWS (similar to docker-compose, but instead of *services*, we will have *container definitions*):
        - We already have the images uploaded to DockerHub, so we are just managing these
        - The documentation we will use is here: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#container_definitions
    - Creating the DB services:
        - **OPTION 1:** The DB storage will be updated in a real production environment:
            - `Redis` should run inside the `AWS ElastiCache` or `EC`
            - `Postgres` should run inside the `AWS Relational Database Service` or `RDS`

            1. Create `RDS`:
                - Create with simple-create
                - Select `Postgres`
                - In `Settings` add a cluster identifier (name), set "self managed" for credentials and set the same password you are using as the env variable
                - Create new SG
                - Database name: fibvalues (this wasn't filled anywhere!)

            2. Create `ElastiCache`: (maybe we need to create a "Global datastore" instead?)
                - Create Redis OSS
                - Design your own / Easy Create
                - Select Demo mode for cheapest version
                - Add name
                - Add sybnet name (eg redis group)
            
            - To establish connection between these services, we need to create a new `Security Group`, that says "allow any traffic from any other AWS service that has the same SG":
                - In VPC dashboard create a new SG:
                    - add name
                    - add inbound rule: Custom TCP / TCP / port 5432-6379 / allow from newly created SG
                - Assign this SH to the EB instance, the RDS and the ED (add as second, don't need to remove the original!)

        - **OPTION 2:** But we will still make our DB services using containers, for practice of Docker!
    
    - Add environmental variables to Elastic Beanstalk:
        - Click on the environment -> Configuration / Configure updates, monitoring, and logging / Platform software  / Environment properties:
            - Add `REDIS_HOST` with elasticache's Primary endpoint as a value, without the port! (As I'm using Redis OSS cache, there's no Primary Endpoint, so I'm using the configuration endpoint)
            - Add `REDIS_PORT` with 6379 (default port)
            - Add `PGUSER` with "postgres" or what you used
            - Add `PGPASSWORD` with "postgres_password" or what you used
            - Add `PGHOST` with RDS / Instances -> click on instance -> under Connectivity & Security you can find the Endpoint to add
            - Add `PGDATABASE` with DB instance ID as a value (same as the name of the RDS you created)
            - Add `PGPORT` with 5432 (default port)
        - **Note:** As opposed to docker-compose, where we add the ENV variables to each service, here the ENV variables added to the Elastic Beanstalk will be shared with all containers listed in the Dockerrun file
    
    - Create access with IAM user:
        - Go to AWS console / IAM:
            - Create a new user with deploy access to Elastic Beanstalk:
                - Click on User -> Create User -> add name like "fibonacci-calculator-deployer" (don't need to add console access!)
                - Attach existing policies -> Add all policies related to "beanstalk" (up to 10 only!)
            - Add Access keys ("other" type -> Don't forget to download .csv file!)
        
        - Got to `travis.ci` site, select the project, and under Settings / Environmental variables add:
            - `AWS_ACCESS_KEY` with the user's access key as a value
            - `AWS_SECRET_KEY` with the user's secret access key as a value
    
    - In `.travis.yml` file:
        - We need to add our AWS credentials (`$AWS_ACCESS_KEY` and `$AWS_SECRET_KEY`)
        - Add `deploy` section:
            
                deploy:
                    provider: elasticbeanstalk
                    region: eu-west-2
                    app: fibonacci-calculator (get this from Elastic Beanstalk / your environment / App name)
                    env: Fibonacci-calculator-env (different than the app name!)
                    bucket_name: elasticbeanstalk-eu-west-2-891376988072 (this gets generated automatically!)
                    bucket_path: fibonacci-calculator (can be same as app name)
                    on:
                        branch: main
                    access_key_id: $AWS_ACCESS_KEY
                    secret_access_key: $AWS_SECRET_KEY