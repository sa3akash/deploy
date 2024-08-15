
Yes, it's possible to deploy an Ubuntu server app and MongoDB using Docker Compose and set up automatic backups to Google Drive. Here's a step-by-step guide to achieving this:

### Step 1: Set Up Docker Compose for Ubuntu Server App and MongoDB

Create a `docker-compose.yml` file to define your Ubuntu server app and MongoDB services:

```yaml
version: '3.8'

services:
  app:
    image: ubuntu:latest
    container_name: ubuntu_server_app
    volumes:
      - ./app:/usr/src/app
    command: /usr/src/app/start.sh
    ports:
      - "8080:80"

  mongo:
    image: mongo:latest
    container_name: mongo_db
    volumes:
      - mongo_data:/data/db
    ports:
      - "27017:27017"

volumes:
  mongo_data:
```

### Step 2: Create a Backup Script

Create a script to back up MongoDB and upload the backup to Google Drive. Save this script as `backup.sh`.

```bash
#!/bin/bash

# Variables
BACKUP_DIR="/backup"
DATE=$(date +"%Y%m%d%H%M")
MONGO_CONTAINER="mongo_db"
BACKUP_NAME="mongo_backup_$DATE.gz"

# Create backup directory if not exists
mkdir -p $BACKUP_DIR

# Dump MongoDB database
docker exec $MONGO_CONTAINER mongodump --archive=$BACKUP_DIR/$BACKUP_NAME --gzip

# Upload to Google Drive (requires rclone configuration)
rclone copy $BACKUP_DIR/$BACKUP_NAME remote:backups/mongo

# Clean up
rm -rf $BACKUP_DIR
```

### Step 3: Configure Rclone for Google Drive

1. Install rclone on your host machine:

```bash
curl https://rclone.org/install.sh | sudo bash
```

2. Configure rclone to work with Google Drive:

```bash
rclone config
```

Follow the prompts to set up Google Drive, and name the remote as `remote`.

### Step 4: Set Up Cron Job for Daily Backups

Set up a cron job to run the backup script daily.

1. Open the crontab editor:

```bash
crontab -e
```

2. Add the following line to schedule the backup script to run every day at midnight:

```bash
0 0 * * * /path/to/backup.sh
```

Replace `/path/to/backup.sh` with the actual path to your `backup.sh` script.

### Step 5: Deploy Docker Compose

Deploy your services using Docker Compose:

```bash
docker-compose up -d
```

### Summary

1. Create a `docker-compose.yml` file for your Ubuntu server app and MongoDB.
2. Write a `backup.sh` script to back up MongoDB and upload to Google Drive.
3. Configure rclone for Google Drive.
4. Set up a cron job to run the backup script daily.
5. Deploy your Docker Compose setup.

This setup ensures that your MongoDB data is backed up daily to Google Drive.
---

To deploy a MERN stack application on an Ubuntu server with Hostinger using Docker Compose and set up automatic daily backups of MongoDB to Google Drive (with old backups being deleted automatically), follow these steps:

### Step 1: Set Up Docker Compose for MERN Stack

Create a `docker-compose.yml` file for your MERN stack application and MongoDB.

```yaml
version: '3.8'

services:
  web:
    image: node:14
    container_name: mern_web
    working_dir: /app
    volumes:
      - ./app:/app
    command: sh -c "cd client && npm install && npm run build && cd ../server && npm install && npm start"
    ports:
      - "3000:3000"
    depends_on:
      - mongo

  mongo:
    image: mongo:latest
    container_name: mongo_db
    volumes:
      - mongo_data:/data/db
    ports:
      - "27017:27017"

volumes:
  mongo_data:
```

### Step 2: Create the Backup Script

Create a script `backup.sh` that will back up MongoDB and upload it to Google Drive using `rclone`. This script will also remove old backups.

```bash
#!/bin/bash

# Variables
BACKUP_DIR="/backup"
DATE=$(date +"%Y%m%d%H%M")
MONGO_CONTAINER="mongo_db"
BACKUP_NAME="mongo_backup_$DATE.gz"
RETENTION_DAYS=7
RCLONE_REMOTE="remote:backups/mongo"

# Create backup directory if not exists
mkdir -p $BACKUP_DIR

# Dump MongoDB database
docker exec $MONGO_CONTAINER mongodump --archive=$BACKUP_DIR/$BACKUP_NAME --gzip

# Upload to Google Drive (requires rclone configuration)
rclone copy $BACKUP_DIR/$BACKUP_NAME $RCLONE_REMOTE

# Clean up local backup
rm -rf $BACKUP_DIR

# Delete old backups from Google Drive
rclone -q --min-age ${RETENTION_DAYS}d delete $RCLONE_REMOTE

# Remove empty directories in Google Drive
rclone -q rmdirs $RCLONE_REMOTE --leave-root
```

### Step 3: Configure Rclone for Google Drive

1. Install `rclone` on your host machine:

```bash
curl https://rclone.org/install.sh | sudo bash
```

2. Configure `rclone` to work with Google Drive:

```bash
rclone config
```

Follow the prompts to set up Google Drive, and name the remote as `remote`.

### Step 4: Set Up Cron Job for Daily Backups

Set up a cron job to run the backup script daily.

1. Open the crontab editor:

```bash
crontab -e
```

2. Add the following line to schedule the backup script to run every day at midnight:

```bash
0 0 * * * /path/to/backup.sh
```

Replace `/path/to/backup.sh` with the actual path to your `backup.sh` script.

### Step 5: Deploy Docker Compose

Deploy your services using Docker Compose:

```bash
docker-compose up -d
```

### Summary

1. **Docker Compose Configuration**: Set up `docker-compose.yml` for the MERN stack app and MongoDB.
2. **Backup Script**: Write a `backup.sh` script to back up MongoDB, upload to Google Drive, and delete old backups.
3. **Rclone Configuration**: Install and configure `rclone` for Google Drive.
4. **Cron Job**: Schedule the backup script to run daily using cron.
5. **Deployment**: Deploy the MERN stack using Docker Compose.

This setup ensures that your MongoDB data is backed up daily to Google Drive, with old backups being automatically deleted after a specified retention period.

---

Restoring your MongoDB data from a backup stored in Google Drive involves a few steps: downloading the backup from Google Drive, and using `mongorestore` to import the data back into your MongoDB instance. Below are the detailed steps to accomplish this.

### Step 1: Download the Backup from Google Drive

Use `rclone` to download the backup file from Google Drive. Create a script called `restore.sh` to automate this process.

```bash
#!/bin/bash

# Variables
BACKUP_DIR="/backup"
BACKUP_NAME="mongo_backup_$1.gz"
RCLONE_REMOTE="remote:backups/mongo"

# Create backup directory if not exists
mkdir -p $BACKUP_DIR

# Download backup from Google Drive
rclone copy $RCLONE_REMOTE/$BACKUP_NAME $BACKUP_DIR

# Extract and restore the backup
docker exec -i mongo_db mongorestore --archive=$BACKUP_DIR/$BACKUP_NAME --gzip

# Clean up local backup
rm -rf $BACKUP_DIR
```

### Step 2: Restore the Backup

To restore a backup, you need to specify the exact backup file name (timestamp) you want to restore. For example, if you want to restore the backup from `202408041200`, you would run:

```bash
./restore.sh 202408041200
```

This script will:

1. Download the backup file from Google Drive using `rclone`.
2. Restore the backup to the `mongo_db` container using `mongorestore`.
3. Clean up the local backup directory.

### Step 3: Manual Restore Steps (if not using the script)

If you prefer to restore the backup manually without using the `restore.sh` script, follow these steps:

1. **Download the Backup:**

```bash
# Create backup directory if not exists
mkdir -p /backup

# Download backup from Google Drive
rclone copy remote:backups/mongo/mongo_backup_202408041200.gz /backup
```

2. **Restore the Backup:**

```bash
# Execute mongorestore inside the mongo container
docker exec -i mongo_db mongorestore --archive=/backup/mongo_backup_202408041200.gz --gzip
```

3. **Clean Up:**

```bash
# Remove the downloaded backup file
rm -rf /backup
```

### Summary

1. **Restore Script**: Create a `restore.sh` script to automate the download and restore process.
2. **Run the Restore Script**: Use the script with the appropriate timestamp to restore a specific backup.
3. **Manual Restore**: Alternatively, perform the steps manually if you prefer more control.

By following these steps, you can restore your MongoDB data from backups stored in Google Drive, ensuring your data can be recovered when needed.
---

To restore MongoDB data on a different server from backups stored in Google Drive, you can follow these steps:

### Step 1: Install Necessary Tools on the New Server

1. **Install Docker and Docker Compose:**

```bash
sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl start docker
sudo systemctl enable docker
```

2. **Install rclone:**

```bash
curl https://rclone.org/install.sh | sudo bash
```

### Step 2: Configure rclone for Google Drive on the New Server

Run `rclone config` and follow the prompts to configure Google Drive, using the same configuration you used on the original server. Name the remote `remote`.

```bash
rclone config
```

### Step 3: Set Up Docker Compose for MongoDB on the New Server

Create a `docker-compose.yml` file to set up MongoDB on the new server:

```yaml
version: '3.8'

services:
  mongo:
    image: mongo:latest
    container_name: mongo_db
    volumes:
      - mongo_data:/data/db
    ports:
      - "27017:27017"

volumes:
  mongo_data:
```

Start the MongoDB container:

```bash
docker-compose up -d
```

### Step 4: Create the Restore Script on the New Server

Create a script called `restore.sh` on the new server to automate the restore process:

```bash
#!/bin/bash

# Variables
BACKUP_DIR="/backup"
BACKUP_NAME="mongo_backup_$1.gz"
RCLONE_REMOTE="remote:backups/mongo"

# Create backup directory if not exists
mkdir -p $BACKUP_DIR

# Download backup from Google Drive
rclone copy $RCLONE_REMOTE/$BACKUP_NAME $BACKUP_DIR

# Extract and restore the backup
docker exec -i mongo_db mongorestore --archive=$BACKUP_DIR/$BACKUP_NAME --gzip

# Clean up local backup
rm -rf $BACKUP_DIR
```

Make the script executable:

```bash
chmod +x restore.sh
```

### Step 5: Restore the Backup on the New Server

Run the `restore.sh` script with the appropriate backup timestamp:

```bash
./restore.sh 202408041200
```

This script will:

1. Download the backup file from Google Drive using `rclone`.
2. Restore the backup to the `mongo_db` container using `mongorestore`.
3. Clean up the local backup directory.

### Summary

1. **Install Tools**: Install Docker, Docker Compose, and `rclone` on the new server.
2. **Configure rclone**: Set up `rclone` for Google Drive.
3. **Docker Compose**: Set up and start a MongoDB container using Docker Compose.
4. **Restore Script**: Create and run a `restore.sh` script to download and restore the backup.

By following these steps, you can restore your MongoDB data on a new server from backups stored in Google Drive.

---

If you want to manually restore MongoDB data on a new server from a backup stored in Google Drive, follow these steps:

### Step 1: Install Necessary Tools on the New Server

1. **Install Docker and Docker Compose:**

```bash
sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl start docker
sudo systemctl enable docker
```

2. **Install rclone:**

```bash
curl https://rclone.org/install.sh | sudo bash
```

### Step 2: Configure rclone for Google Drive

Run `rclone config` and follow the prompts to configure Google Drive. Name the remote `remote`.

```bash
rclone config
```

### Step 3: Set Up Docker Compose for MongoDB

Create a `docker-compose.yml` file to set up MongoDB:

```yaml
version: '3.8'

services:
  mongo:
    image: mongo:latest
    container_name: mongo_db
    volumes:
      - mongo_data:/data/db
    ports:
      - "27017:27017"

volumes:
  mongo_data:
```

Start the MongoDB container:

```bash
docker-compose up -d
```

### Step 4: Manually Download the Backup from Google Drive

1. Create a directory to store the backup:

```bash
mkdir -p /backup
```

2. Use `rclone` to download the backup from Google Drive:

```bash
rclone copy remote:backups/mongo/mongo_backup_202408041200.gz /backup
```

### Step 5: Restore the Backup

1. Run the `mongorestore` command inside the MongoDB container to restore the backup:

```bash
docker exec -i mongo_db mongorestore --archive=/backup/mongo_backup_202408041200.gz --gzip
```

### Step 6: Clean Up

1. Remove the downloaded backup file:

```bash
rm -rf /backup
```

### Summary

1. **Install Tools**: Install Docker, Docker Compose, and `rclone` on the new server.
2. **Configure rclone**: Set up `rclone` for Google Drive.
3. **Docker Compose**: Set up and start a MongoDB container using Docker Compose.
4. **Download Backup**: Use `rclone` to download the backup from Google Drive.
5. **Restore Backup**: Use `mongorestore` to restore the backup inside the MongoDB container.
6. **Clean Up**: Remove the downloaded backup file.

By following these steps, you can manually restore your MongoDB data on a new server from backups stored in Google Drive.