# 🚀 Day 11/90 — Docker Storage & Networking

**DevOps Winter Arc | Docker Week | Learn • Practice • Build**

This hands-on project explores Docker storage, PostgreSQL data persistence, container networking, DNS resolution, port mapping, and backup practices using Docker Desktop and Windows PowerShell.

## 📌 Project Overview

A database must retain its data even when its container is removed and recreated. In this project, I deployed PostgreSQL with a named Docker volume, connected a separate container through a custom bridge network, tested DNS behavior, diagnosed a host-port conflict, and created a compressed volume backup.

## 🎯 Objectives

* Understand Docker container writable layers versus named volumes.
* Create and inspect Docker volumes and custom bridge networks.
* Deploy PostgreSQL with persistent storage.
* Prove data persistence after container recreation.
* Connect containers using Docker's internal DNS.
* Compare name resolution on the default and custom bridge networks.
* Troubleshoot host-port allocation conflicts.
* Create and inspect a compressed PostgreSQL volume archive.
* Document results using terminal screenshots and reproducible commands.

## 🛠️ Technologies Used

* Docker Desktop
* Docker CLI
* Windows PowerShell
* PostgreSQL 16 Alpine
* NGINX Alpine
* Alpine Linux
* Docker bridge networking
* Docker named volumes

## 🏗️ Architecture

```text
                    Docker Host
                         |
             +-----------+-----------+
             |                       |
         app-net                  Default Bridge
      (User-defined)             (DNS comparison)
             |
       +-----+------+
       |            |
  PostgreSQL     Temporary
  Container db   Client Container
       |
   Named Volume
     pgdata
       |
  Database Files

  NGINX w1                 NGINX w2
  Host: 18080              Host: 18081
  Container: 80            Container: 80

  pgdata
     |
  Compressed Backup
  pgdata-backup.tgz
```

## 📂 Project Structure

```text
day11-storage-networking/
├── README.md
├── .gitignore
├── pgdata-backup.tgz       # Local backup; excluded from Git
└── evidence/
    ├── 01-network-volume.png
    ├── 02-postgres-running.png
    ├── 03-data-before-recreation.png
    ├── 04-data-persistence.png
    ├── 05-container-networking.png
    ├── 06-dns-test.png
    ├── 07-port-conflict-and-fix.png
    └── 08-volume-backup.png
```

## ⚙️ Implementation

### 1. Create the network and volume

```powershell
docker network create app-net
docker volume create pgdata

docker network inspect app-net
docker volume inspect pgdata
```

**Learning:** A user-defined bridge network enables container communication and automatic name-based DNS resolution. A named volume stores data independently of a container's writable layer.

### 2. Deploy PostgreSQL

```powershell
docker run -d --name db --network app-net `
  -e POSTGRES_USER=postgres `
  -e POSTGRES_PASSWORD=changeme `
  -e POSTGRES_DB=shop `
  -v pgdata:/var/lib/postgresql/data `
  postgres:16-alpine
```

Verify the container and database readiness:

```powershell
docker ps
docker exec db pg_isready -U postgres -d shop
```

**Learning:** The `-v` option mounts the existing named volume at PostgreSQL's data directory. The database can reuse this storage after container replacement.

> **Security note:** `changeme` is a lab-only password. Never use this example credential in a production environment.

### 3. Create and insert database data

Create a table:

```powershell
docker exec db psql -U postgres -d shop -c "CREATE TABLE IF NOT EXISTS notes (id SERIAL PRIMARY KEY, message TEXT);"
```

Insert a sample record:

```powershell
docker exec db psql -U postgres -d shop -c "INSERT INTO notes (message) VALUES ('I survive!');"
```

Query the table:

```powershell
docker exec db psql -U postgres -d shop -c "SELECT * FROM notes;"
```

### 4. Verify data persistence

Remove the container without deleting the volume:

```powershell
docker rm -f db
```

Recreate PostgreSQL using the same `pgdata` volume by repeating the PostgreSQL deployment command.

Verify readiness and query the original data:

```powershell
docker exec db pg_isready -U postgres -d shop

docker exec db psql -U postgres -d shop -c "SELECT * FROM notes;"
```

**Expected result:** The original `I survive!` record remains available after the container is recreated.

**Learning:** Container removal does not remove a named volume unless the volume itself is explicitly deleted or otherwise removed.

### 5. Test container networking

Connect a temporary PostgreSQL client container to `app-net`:

```powershell
docker run --rm --network app-net `
  -e PGPASSWORD=changeme `
  postgres:16-alpine `
  psql -h db -U postgres -d shop -c "SELECT * FROM notes;"
```

**Expected result:** The client resolves `db`, connects to PostgreSQL, and retrieves the existing record.

**Learning:** Containers on a user-defined bridge network can use Docker's embedded DNS to discover one another by name.

### 6. Compare DNS behavior

Test the default bridge:

```powershell
docker run --rm alpine:3.20 nslookup db
```

Test the custom bridge:

```powershell
docker run --rm --network app-net alpine:3.20 nslookup db
```

**Expected behavior:**

* Default bridge: the container name normally does not resolve automatically.
* User-defined bridge: the name `db` should resolve to the database container's IP address.

The actual DNS output depends on the Docker environment.

### 7. Troubleshoot port mapping

Start the first NGINX container:

```powershell
docker run -d --name w1 -p 18080:80 nginx:alpine
curl.exe -I http://localhost:18080
```

Attempt to reuse host port `18080`:

```powershell
docker run -d --name w2 -p 18080:80 nginx:alpine
```

The second command should fail with a host-port allocation conflict if `w1` already owns that port.

Fix the conflict by publishing the second container on `18081`:

```powershell
docker rm -f w2
docker run -d --name w2 -p 18081:80 nginx:alpine
curl.exe -I http://localhost:18081
```

**Expected result:** Both NGINX containers can serve HTTP using different host ports.

**Learning:** Multiple containers can listen on the same internal port, but they cannot simultaneously bind the same host IP and port combination.

### 8. Back up the named volume

Stop PostgreSQL:

```powershell
docker stop db
```

Create a compressed archive in the project directory:

```powershell
docker run --rm `
  -v pgdata:/data `
  -v "${PWD}:/backup" `
  alpine:3.20 `
  tar czf /backup/pgdata-backup.tgz -C /data .
```

Inspect the backup file:

```powershell
Get-Item .\pgdata-backup.tgz |
  Select-Object Name, Length, LastWriteTime
```

List the archive contents:

```powershell
docker run --rm `
  -v "${PWD}:/backup" `
  alpine:3.20 `
  tar -tzf /backup/pgdata-backup.tgz
```

Restart PostgreSQL and verify readiness:

```powershell
docker start db
docker exec db pg_isready -U postgres -d shop
```

**Learning:** A named volume provides persistent live storage, while a backup archive provides a separate copy intended for recovery. A production backup should also be restored into a separate test environment to verify recoverability.

## 📸 Evidence and Results

Screenshots are stored in the `evidence/` directory.

| Screenshot                      | Evidence                                       |
| ------------------------------- | ---------------------------------------------- |
| `01-network-volume.png`         | Network and volume creation and inspection     |
| `02-postgres-running.png`       | PostgreSQL container and readiness             |
| `03-data-before-recreation.png` | Database record before removal                 |
| `04-data-persistence.png`       | Record after container recreation              |
| `05-container-networking.png`   | Client query using hostname `db`               |
| `06-dns-test.png`               | Default versus custom bridge DNS               |
| `07-port-conflict-and-fix.png`  | Port conflict and successful alternative port  |
| `08-volume-backup.png`          | Backup archive, size, and restart verification |

Screenshots should contain actual terminal results. Replace or update this evidence if any test has not yet been completed.

## 🧠 Key Learnings

* Named volumes separate persistent data from a container's writable layer.
* A container can be recreated without losing data when it reuses the same volume.
* User-defined bridge networks provide automatic container-name resolution.
* Host-port conflicts can be resolved by selecting another available host port.
* File-level backups should be created from a cleanly stopped database for this exercise.
* A backup should be tested through restoration, not merely by checking that the archive exists.

## 🔍 Troubleshooting

| Problem                     | Diagnostic or solution                                                   |
| --------------------------- | ------------------------------------------------------------------------ |
| Docker daemon unavailable   | Start Docker Desktop and verify `docker version`.                        |
| Invalid reference format    | Use PowerShell backticks for multiline commands, not CMD `^` characters. |
| Container not found         | Check `docker ps -a` and create the container if needed.                 |
| PostgreSQL not ready        | Check `docker logs db` and retry `pg_isready`.                           |
| Database record missing     | Confirm that the same `pgdata` volume was reused.                        |
| DNS resolution fails        | Confirm that both containers are attached to `app-net`.                  |
| Host port already allocated | Use a different host port, such as `18081`.                              |
| Backup archive missing      | Check the current directory and the output of `docker run`.              |

## 🔐 Security and Cleanup

* Do not publish production credentials in source code.
* Keep the database backup out of public Git repositories.
* Do not remove `pgdata` until persistence testing and backup verification are complete.
* Remove only temporary containers when cleanup is appropriate.
* Test database restoration before depending on a backup.

## 🚀 Future Improvements

* Restore the backup into a separate PostgreSQL test environment.
* Use Docker secrets or a secure credential management solution.
* Add automated backup verification.
* Implement PostgreSQL-aware logical backups with `pg_dump`.
* Add health checks and resource monitoring.
* Document recovery time and recovery point objectives.

## ✅ Conclusion

This project demonstrates core Docker storage and networking concepts through a PostgreSQL persistence test, custom bridge networking, DNS resolution, port conflict troubleshooting, and volume backup.

It provides hands-on practice with container lifecycle management and data protection—essential skills for containerized application operations.

**Challenge:** DevOps Winter Arc — Day 11/90
**Focus:** Docker Storage & Networking
**Environment:** Windows PowerShell + Docker Desktop
