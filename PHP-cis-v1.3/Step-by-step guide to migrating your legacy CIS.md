Step-by-step guide to migrating your legacy Computer Inventory & Ticketing System (`CIS v1.3`) from the remote Windows Server to your local Windows 11 WSL2 environment.

This plan is grouped by specific tasks to keep it manageable.

### Phase 1: Data Retrieval & Migration

*Goal: Get the code and database off the old server and onto your laptop safely.*

**Task 1.1: Copy the Source Code**
Since the source is on a network share (`\\thbkk0824\e$\...`), the safest way is to copy it to a local folder first.

1. On your Windows 11 Laptop, create a folder: `C:\Projects\CIS_Migration`.
2. Open File Explorer and navigate to `\\thbkk0824\e$\wwwroot\CIS\v1.3`.
3. Copy **all files** from the server to `C:\Projects\CIS_Migration`.

**Task 1.2: Export the Database**
You need the raw data (Inventory items, Tickets, Users).

1. Log in to the Windows Server (`thbkk0824`) or use a tool like **HeidiSQL** / **phpMyAdmin** if accessible.
2. Export the database to a `.sql` file (e.g., `cis_backup.sql`).
3. **Crucial:** Ensure you export "Structure and Data".
4. Move this `cis_backup.sql` file into your `C:\Projects\CIS_Migration` folder.

---

### Phase 2: WSL & Filesystem Setup

*Goal: Move files into the Linux filesystem for speed (Docker on Windows runs 10x faster when files are inside Ubuntu, not on C: drive).*

**Task 2.1: Open Ubuntu**
Open your **Ubuntu** terminal.

**Task 2.2: Create Project Directory in Linux**
Run these commands to create a folder and move into it:

```bash
mkdir -p ~/projects/cis_legacy
cd ~/projects/cis_legacy

```

**Task 2.3: Import Files from Windows to Linux**
Copy the files from your Windows C: drive into this Linux folder.

```bash
# Copy files from Windows C: drive to current Linux folder
cp -r /mnt/c/Projects/CIS_Migration/* .

```

* `ls -l` to verify you see your PHP files and the `.sql` file.

---

### Phase 3: Docker Containerization

*Goal: Create the "Time Capsule" environment so the old PHP 5.6 code runs.*

**Task 3.1: Create the Configuration Files**
Inside `~/projects/cis_legacy`, create two files.

**File 1: `Dockerfile**` (Fixes PHP 5.6 extensions)

```dockerfile
FROM php:5.6-apache

# Point to archived Debian repos for legacy support
RUN echo "deb http://archive.debian.org/debian stretch main" > /etc/apt/sources.list \
    && echo "deb http://archive.debian.org/debian-security stretch/updates main" >> /etc/apt/sources.list

# Install specific extensions likely needed for Inventory/Tickets (GD, MySQL)
RUN apt-get update && apt-get install -y libpng-dev \
    && docker-php-ext-install gd mysqli pdo pdo_mysql mysql \
    && docker-php-ext-enable mysqli

# Enable Apache Rewrite (common for routing)
RUN a2enmod rewrite

```

*(Note: I added `gd` and `mysql` as older inventory systems often use GD for barcodes/images and `mysql_` functions).*

**File 2: `docker-compose.yml**`

```yaml
version: '3.8'

services:
  app:
    build: .
    container_name: cis_app
    ports:
      - "8080:80"
    volumes:
      - .:/var/www/html
    depends_on:
      - db
    networks:
      - cis_net

  db:
    image: mysql:5.7
    container_name: cis_db
    platform: linux/x86_64
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: cis_db
      MYSQL_USER: cis_user
      MYSQL_PASSWORD: cis_pass
    volumes:
      # Mount the SQL dump to auto-import on first start
      - ./cis_backup.sql:/docker-entrypoint-initdb.d/init.sql
      - db_data:/var/lib/mysql
    networks:
      - cis_net

networks:
  cis_net:

volumes:
  db_data:

```

**Task 3.2: Rename SQL Dump**
Rename your database export file to `cis_backup.sql` so the Docker configuration above finds it automatically.

```bash
mv *.sql cis_backup.sql

```

**Task 3.3: Start the Engine**

```bash
docker-compose up -d --build

```

* Wait for it to build. If successful, visiting `http://localhost:8080` in Windows Chrome should show your login page (or an error, which is progress!).

---

### Phase 4: Code Remediation

*Goal: Connect the App to the Database.*

**Task 4.1: Find the Database Connection File**
Look for a file named `config.php`, `db.php`, `connect.php`, or similar.

```bash
grep -r "mysql_connect" .

```

*(This command searches all files for database connection code).*

**Task 4.2: Update Credentials**
Edit that file (you can use `code .` in Ubuntu to open VS Code on Windows editing these Linux files). Update the settings to match `docker-compose.yml`:

* **Host:** `db` (or `cis_db`)
* **User:** `cis_user`
* **Pass:** `cis_pass`
* **DB Name:** `cis_db`

**Task 4.3: Fix "Short Tags" (Optional)**
If you see PHP code on the screen instead of the app, open `php.ini` (or add to Dockerfile) to set `short_open_tag=On`.

---

### Phase 5: Preparation for AI Enhancement

*Goal: Modernize the app without rewriting it entirely.*

Now that it is running locally, you don't want to write complex AI logic in PHP 5.6. Instead, build a **Sidecar AI Service**.

**Task 5.1: Create an AI Service (Python)**
Add a Python container to your `docker-compose.yml`. This will handle the "Brain" work.

```yaml
  ai_service:
    image: python:3.9-slim
    container_name: cis_ai
    volumes:
      - ./ai:/app
    command: python app.py
    networks:
      - cis_net

```

**Task 5.2: Identify AI Use Cases**
For a ticketing/inventory system, here are the best quick-win features:

1. **Smart Tagging:** When a user submits a ticket ("My screen is flickering"), the AI (Python) reads the database, tags it as "Hardware -> Monitor", and assigns priority.
2. **Inventory Forecasting:** Predict when printer toner or spare laptops will run out based on historical usage in MySQL.
3. **Chatbot:** A "Helpdesk Assistant" that queries the DB to answer "Do we have any MacBook Pros in stock?"

**Next Step:**
Would you like me to generate the **Python script** for the "Smart Tagging" feature to demonstrate how the legacy PHP app can talk to the new AI service?