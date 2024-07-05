# fibonacci-calculator
- A simple fibonacci sequence calculator application created to showcase complex application deployment with automated CI/CD integration

# guide
- Arhitecture in video 116

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
    - Create postgres service (potential issue with image version!)
    - Create redis service (potential issue with image version!)
    - Create server service:
        - build from Dockerfile
        - add volumes:
            - tag `/app/node_modules` folder, so that is not overwritten!
            - reference `./server:/app`, so that the `app` folder in the `server` container is always what is in our machine's `server` folder
        - specify environmental variables:
            - `variableName=value` (sets variable at runtime) - USE THIS METHOD
            - `variableName` (in this case the value is taken from my local environment) - it's still set at runtime
            - potential issue - rename variables from `PGPASSWORD` to postgres documentation format `POSTGRES_PASSWPRD`
