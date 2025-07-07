# develop-a-secure-SIEM
 1. Download and Install Winlogbeat
 2.  Step 1: Download Winlogbeat
Go to:
https://www.elastic.co/downloads/beats/winlogbeat
Choose the Windows .zip version (e.g., winlogbeat-9.0.0-windows-x86_64.zip)
Extract the ZIP to:
C:\Program Files\Winlogbeat
After extraction, your folder should look like:
C:\Program Files\Winlogbeat\winlogbeat-9.0.0-windows-x86_64\
2. Configure Winlogbeat to Collect Logs
Step 2: Edit winlogbeat.yml
Open the file:
C:\Program Files\Winlogbeat\winlogbeat-9.0.0-windows-x86_64\winlogbeat.yml
###################### Winlogbeat Configuration ########################

winlogbeat.event_logs:
  - name: Application
    ignore_older: 72h
  - name: System
  - name: Security
  - name: Microsoft-Windows-Sysmon/Operational
  - name: Windows PowerShell
    event_id: 400, 403, 600, 800
  - name: Microsoft-Windows-PowerShell/Operational
    event_id: 4103, 4104, 4105, 4106
  - name: ForwardedEvents
    tags: [forwarded]

setup.template.settings:
  index.number_of_shards: 1

setup.kibana:
  host: "http://localhost:5601/"

# ================================ OUTPUT ================================

# Disable Elasticsearch output
# output.elasticsearch:
#   hosts: ["http://localhost:9200"]

# Enable Logstash output
output.logstash:
  hosts: ["localhost:5044"]

# ================================ PROCESSORS ================================

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~

# ================================ LOGGING ================================

logging.level: info
logging.to_files: true
logging.files:
  path: "C:/Program Files/Winlogbeat/winlogbeat-9.0.0-windows-x86_64/logs"
  name: winlogbeat
  keepfiles: 7
  permissions: 0644

# ============================ MONITORING ============================

#monitoring.enabled: false


 3. Run Winlogbeat (PowerShell Commands)
 4.  Step 3: Open PowerShell as Administrator
Click Start → Search “PowerShell” → Right-click → Run as Administrator

Step 4: Navigate to Winlogbeat folder
cd "C:\Program Files\Winlogbeat\winlogbeat-9.0.0-windows-x86_64"

type commands
Start-Service winlogbeat
To stop the service:
Stop-Service winlogbeat
6. View Logs Collected by Winlogbeat
Winlogbeat creates logs in:
C:\Program Files\Winlogbeat\winlogbeat-9.0.0-windows-x86_64\logs
To view logs in PowerShell:
Get-Content .\winlogbeat-20250612.ndjson | ForEach-Object { $_ | ConvertFrom-Json }
Step 1: Download Docker Desktop
  https://www.docker.com/products/docker-desktop
Click "Download for Windows (Docker Desktop)"
set up docker and now open docker terminal
Step 2: Verify Docker Installation
Open Command Prompt or PowerShell and run:
docker --version
now configure elasticsearch,kibana,logstash in docker
docker-compose.yml code
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - xpack.security.transport.ssl.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:8.12.0
    container_name: logstash
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
      - ./data:/data
    ports:
      - "5000:5000"
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

volumes:
  esdata:


  logstash.conf file
  input {
  file {
    path => "/data/winlogbeat.ndjson"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}

filter {
  grok {
    match => { "message" => "%{COMMONAPACHELOG}" }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "wing-logs"
  }
  stdout { codec => rubydebug }
}


3. Run Docker Compose
Now run this command to start all containers in detached mode:
docker-compose up -d
What this does:
Downloads the ELK stack images if not already present

Starts the containers:

Elasticsearch (port 9200)

Logstash (port 5044)

Kibana (port 5601)

Runs in the background (detached mode -d)

✅ 4. Verify Containers Are Running

docker ps
You should see something like:

arduino
Copy
Edit
CONTAINER ID   IMAGE              STATUS          PORTS
abc12345       kibana:8.x         Up 1 min        5601/tcp
def67890       elasticsearch:8.x  Up 1 min        9200/tcp
ghi90123       logstash:8.x       Up 1 min        5044/tcp
🧠 Tips:
If this is your first run, it may take a few minutes to pull the images.

To view logs (useful for debugging):
docker-compose logs -f
To stop everything:
docker-compose down

Step 1: Access Kibana in Browser
Open your browser and go to:

http://localhost:5601
If you're on a different host, replace localhost with the server IP address.

 Step 2: Log in to Kibana
Username: elastic
Password: (defined in .env or docker-compose.yml, often changeme if not secured)

If you don't know the password, check your .env file or run:

bash
Copy
Edit
docker logs <elasticsearch_container_id>
Look for Generated password for the elastic user.

Step 3: Create Index Pattern for Winlogbeat
To view Windows logs sent by Winlogbeat:

Go to Kibana sidebar → Stack Management

Click "Index Patterns" → "Create index pattern"

Enter:

Copy
Edit
winlogbeat-*
Choose @timestamp as the time field

Click Create index pattern

Now Kibana will recognize your incoming logs.

 Step 4: View Logs in Discover
Go to "Discover" in the Kibana sidebar

Select the index pattern: winlogbeat-*

You’ll see event logs.

