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