# Run the Tic Tac Toe App and MongoDB with Docker Compose

## Aim

The aim of this task was to use Docker Compose to run two containers together:

- A MongoDB database container
- A Tic Tac Toe Node.js app container

The app needs MongoDB to be running before it can connect to the database.

---

## Images Used

For the app, I used the image I already pushed to Docker Hub:

```bash
monty97/tech610-tttapp:1.2.0
```

For MongoDB, I used:

```bash
mongo:8.2.5
```

---

# Manual Version

I first created a folder for the manual version:

```bash
mkdir docker-compose-ttt-manual
cd docker-compose-ttt-manual
touch compose.yaml
```

My `compose.yaml` file:

```yaml
services:
  db:
    image: mongo:8.2.5
    container_name: ttt-mongodb

    ports:
      - "27017:27017"

    volumes:
      - mongo-data:/data/db

    healthcheck:
      test:
        [
          "CMD",
          "mongosh",
          "--quiet",
          "--eval",
          "db.adminCommand('ping').ok"
        ]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s

  app:
    image: monty97/tech610-tttapp:1.2.0
    container_name: ttt-app

    ports:
      - "3000:3000"

    environment:
      MONGODB_URI: mongodb://db:27017/tictactoe

    depends_on:
      db:
        condition: service_healthy

volumes:
  mongo-data:
```

## Explanation

The `db` service creates the MongoDB container.

The database uses port `27017`.

The named volume stores the database data so it is not lost when the container is restarted or removed.

The `app` service uses my Tic Tac Toe image.

The app uses port `3000`.

The environment variable connects the app to MongoDB:

```yaml
MONGODB_URI: mongodb://db:27017/tictactoe
```

I used `db` instead of an IP address because `db` is the name of the MongoDB service in the Compose file.

The health check makes sure MongoDB is ready before the app starts.

---

## Run the Containers

I ran:

```bash
docker compose up -d
```

Then checked both containers:

```bash
docker compose ps
```

Both containers were running and MongoDB showed as healthy.

I could access the app at:

```text
http://localhost:3000
```

---

# Manual Database Seeding

To seed the database manually, I entered the app container:

```bash
docker exec -it ttt-app sh
```

Then I ran:

```bash
node /app/seeds/seed.js
```

The database was seeded with 10 records.

I then exited the container:

```bash
exit
```

I used this method because Git Bash changed the Linux path when I tried to run the command directly.

The original error showed:

```text
Cannot find module '/app/C:/Program Files/Git/app/seeds/seed.js'
```

Entering the container first stopped Git Bash from changing the path.

---

# Testing the Volume

I stopped and removed the containers:

```bash
docker compose down
```

Then started them again:

```bash
docker compose up -d
```

The database data was still there because the MongoDB volume was kept.

I checked the volumes with:

```bash
docker volume ls
```

The volume was called:

```text
docker-compose-ttt-manual_mongo-data
```

To remove the containers and the volume, I used:

```bash
docker compose down -v
```

This deleted the database data as well.

---

# Automatic Seeding Version

I created another folder:

```bash
mkdir docker-compose-ttt-auto-seed
cd docker-compose-ttt-auto-seed
touch compose.yaml
```

My automatic seeding Compose file:

```yaml
services:
  db:
    image: mongo:8.2.5
    container_name: ttt-mongodb-auto

    ports:
      - "27017:27017"

    volumes:
      - mongo-data:/data/db

    healthcheck:
      test:
        [
          "CMD",
          "mongosh",
          "--quiet",
          "--eval",
          "db.adminCommand('ping').ok"
        ]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s

  app:
    image: monty97/tech610-tttapp:1.2.0
    container_name: ttt-app-auto

    ports:
      - "3000:3000"

    environment:
      MONGODB_URI: mongodb://db:27017/tictactoe

    depends_on:
      db:
        condition: service_healthy

    command: >
      sh -c "node /app/seeds/seed.js &&
             node /app/index.js"

volumes:
  mongo-data:
```

The command runs the seed file first, then starts the app:

```yaml
command: >
  sh -c "node /app/seeds/seed.js &&
         node /app/index.js"
```

I ran:

```bash
docker compose up -d
```

Then checked the logs:

```bash
docker compose logs -f app
```

The logs showed:

```text
Seeded MongoDB scoreboard_entries directly via MONGODB_URI (10 records).
Server running at http://localhost:3000
```

This confirmed that the database was seeded automatically.

---

# Seeder Service Version

I also created a separate seeder service.

This is a cleaner method because the seeding is handled by its own container instead of being part of the app startup command.

I created another folder:

```bash
mkdir docker-compose-ttt-seeder-service
cd docker-compose-ttt-seeder-service
touch compose.yaml
```

My Compose file:

```yaml
services:
  db:
    image: mongo:8.2.5
    container_name: ttt-mongodb-seeder

    ports:
      - "27017:27017"

    volumes:
      - mongo-data:/data/db

    healthcheck:
      test:
        [
          "CMD",
          "mongosh",
          "--quiet",
          "--eval",
          "db.adminCommand('ping').ok"
        ]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s

  seed:
    image: monty97/tech610-tttapp:1.2.0
    container_name: ttt-seeder

    environment:
      MONGODB_URI: mongodb://db:27017/tictactoe

    depends_on:
      db:
        condition: service_healthy

    command: ["node", "/app/seeds/seed.js"]

    restart: "no"

  app:
    image: monty97/tech610-tttapp:1.2.0
    container_name: ttt-app-seeder

    ports:
      - "3000:3000"

    environment:
      MONGODB_URI: mongodb://db:27017/tictactoe

    depends_on:
      seed:
        condition: service_completed_successfully

volumes:
  mongo-data:
```

The `seed` service waits for MongoDB, runs the seed file, then exits.

The app only starts after the seed service completes successfully.

---

# Challenge I Faced

When I started the seeder service version, I got this error:

```text
Bind for 0.0.0.0:27017 failed: port is already allocated
```

This happened because the automatic seeding version was still running and already using port `27017`.

I fixed it by stopping the previous Compose project:

```bash
cd ../docker-compose-ttt-auto-seed
docker compose down
```

Then I went back to the seeder service folder and started it again:

```bash
cd ../docker-compose-ttt-seeder-service
docker compose up -d
```

Another option would be to remove the MongoDB port mapping because the app can connect to MongoDB internally using the service name `db`.

---

# Useful Commands

Start the containers:

```bash
docker compose up -d
```

Check the containers:

```bash
docker compose ps
```

Check all containers, including stopped containers:

```bash
docker compose ps -a
```

View logs:

```bash
docker compose logs
```

View app logs:

```bash
docker compose logs app
```

Stop and remove containers:

```bash
docker compose down
```

Stop containers and remove the volume:

```bash
docker compose down -v
```

Enter the app container:

```bash
docker exec -it ttt-app sh
```

Seed the database manually:

```bash
node /app/seeds/seed.js
```

---

# What I Learnt

From this task I learnt:

- How to run multiple containers using Docker Compose
- How containers communicate using service names
- How to pass environment variables into a container
- How to make the app wait for MongoDB
- How to seed a database manually
- How to seed a database automatically
- How Docker volumes keep database data
- How port conflicts happen when two containers use the same host port
