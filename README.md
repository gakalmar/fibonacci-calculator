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

# guide
- Worker folder (calculate number based on index and connect to redis)
    - package.json
    - keys.js
    - index.js

- Server folder (connect ro postgres and create table, connect to redis, express routes)
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
