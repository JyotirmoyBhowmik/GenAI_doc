Step-by-Step Installation and Configuration Guide: AI Document Processing System on RHEL
This comprehensive guide provides detailed instructions for installing and configuring a complete AI document processing system using Ollama, Unstructured.io/Docling, Apache Kafka, PostgreSQL, and web interfaces on Red Hat Enterprise Linux (RHEL).
System Overview
Our implementation will include:
    • LLM Runtime: Ollama for local language model inference
    • Document Processing: Unstructured.io and Docling for multi-format document extraction
    • Database: PostgreSQL for structured data storage
    • Message Broker: Apache Kafka for event streaming
    • Object Storage: MinIO for file storage
    • Caching: Redis for performance optimization
    • Web Interfaces: FastAPI backend + Streamlit frontend
    • Reverse Proxy: Nginx for load balancing and SSL termination
Prerequisites and System Preparation
Step 1: Update System and Install Core Dependencies
# Update system packages
sudo dnf update -y

# Enable EPEL repository for additional packages
sudo dnf install -y epel-release

# Install essential development tools
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y curl wget git nano vim unzip tar gzip \
    gcc gcc-c++ make cmake pkg-config \
    openssl-devel libffi-devel zlib-devel \
    bzip2-devel readline-devel sqlite-devel \
    xz-devel ncurses-devel

# Install Java 17 (required for Kafka)
sudo dnf install -y java-17-openjdk java-17-openjdk-devel

# Set JAVA_HOME environment variable
echo 'export JAVA_HOME=/usr/lib/jvm/java-17-openjdk' | sudo tee -a /etc/environment
source /etc/environment

# Install Python 3.11 and development tools
sudo dnf install -y python3.11 python3.11-devel python3.11-pip 

#python3.11-venv

python3.11 -m venv myenv
source myenv/bin/activate
pip install --upgrade pip


# Install system dependencies for document processing
sudo dnf install -y tesseract tesseract-devel tesseract-langpack-eng \
    leptonica-devel poppler-utils libreoffice pandoc \
    file-devel libmagic-devel

sudo dnf config-manager --set-enabled powertools || sudo dnf config-manager --set-enabled crb
sudo dnf install epel-release


sudo dnf install -y tesseract tesseract-devel  leptonica-devel poppler-utils libreoffice pandoc     file-devel 


# Install media processing libraries
sudo dnf install -y  ImageMagick ghostscript  

#ffmpeg ---- system does not have ffmpeg in any enabled repositories by default.

sudo dnf install -y epel-release
sudo dnf install -y https://mirrors.rpmfusion.org/free/el/rpmfusion-free-release-8.noarch.rpm
sudo dnf install -y https://mirrors.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-8.noarch.rpm

#recheck the URL

sudo dnf config-manager --set-enabled powertools
sudo dnf install -y dnf-plugins-core
sudo dnf install -y ffmpeg


# Create system user for applications
sudo useradd -r -m -s /bin/bash -d /opt/ai-system ai-system

Step 2: Configure Python Virtual Environment
# Create project directory structure
sudo mkdir -p /opt/ai-system/{apps,data,logs,config,models,temp}
sudo chown -R ai-system:ai-system /opt/ai-system

# Switch to ai-system user
sudo su - ai-system

# Create Python virtual environment
cd /opt/ai-system
python3.11 -m venv venv

# Activate virtual environment
source venv/bin/activate

# Upgrade pip and install core packages
pip install --upgrade pip setuptools wheel

# Install Python dependencies for AI document processing
pip install fastapi[all] uvicorn streamlit pandas numpy requests \
    python-multipart aiofiles python-docx psycopg2-binary \
    kafka-python redis langchain sentence-transformers \
    ollama-python qdrant-client minio python-dotenv \
    asyncio pydantic jinja2 python-jose[cryptography] \
    passlib[bcrypt] python-multipart

# Error will be faced during that

# Use Below commands

pip install fastapi[all]
pip install uvicorn
pip install streamlit
pip install pandas
pip install numpy
pip install requests
pip install python-multipart
pip install aiofiles
pip install python-docx
pip install psycopg2-binary
pip install kafka-python
pip install redis
pip install langchain
pip install sentence-transformers
pip install ollama-python
pip install qdrant-client
pip install minio
pip install python-dotenv
pip install asyncio
pip install pydantic
pip install jinja2
pip install python-jose[cryptography]
pip install passlib[bcrypt]

Component Installation and Configuration
Step 3: Install and Configure PostgreSQL Database
Installation
# Exit ai-system user to become root
exit

# Add PostgreSQL repository
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Disable built-in PostgreSQL module
sudo dnf -qy module disable postgresql

# Install PostgreSQL 16
sudo dnf install -y postgresql16-server postgresql16-contrib

# Initialize PostgreSQL database
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb

# Enable and start PostgreSQL service
sudo systemctl enable postgresql-16 --now

# Check service status
sudo systemctl status postgresql-16

Configuration
# Set password for postgres user
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'SecurePassword123';"

# Create application database and user
sudo -u postgres createdb ai_document_db
sudo -u postgres psql -c "CREATE USER ai_user WITH PASSWORD 'AiUserPassword123';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE ai_document_db TO ai_user;"
sudo -u postgres psql -c "ALTER USER ai_user CREATEDB;"

# Configure PostgreSQL for remote connections (if needed)
sudo nano /var/lib/pgsql/16/data/postgresql.conf
# Uncomment and modify: listen_addresses = '*'
# shared_buffers = 256MB
# effective_cache_size = 1GB
# work_mem = 4MB

sudo nano /var/lib/pgsql/16/data/pg_hba.conf
# Add line for local network: host all all 127.0.0.1/32 md5

# Restart PostgreSQL
sudo systemctl restart postgresql-16

# Create application schema
sudo -u postgres psql -d ai_document_db << 'EOF'
CREATE TABLE IF NOT EXISTS processed_documents (
    id SERIAL PRIMARY KEY,
    object_name VARCHAR(255) UNIQUE NOT NULL,
    original_filename VARCHAR(255),
    content_type VARCHAR(100),
    file_size INTEGER,
    extracted_text TEXT,
    ai_summary TEXT,
    metadata JSONB,
    processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS chat_sessions (
    id SERIAL PRIMARY KEY,
    session_id UUID DEFAULT gen_random_uuid(),
    user_query TEXT NOT NULL,
    ai_response TEXT NOT NULL,
    model_used VARCHAR(50),
    response_time_ms INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_processed_documents_object_name ON processed_documents(object_name);
CREATE INDEX idx_processed_documents_processed_at ON processed_documents(processed_at DESC);
CREATE INDEX idx_chat_sessions_created_at ON chat_sessions(created_at DESC);
EOF

Step 4: Install and Configure Apache Kafka
Installation
# Create kafka user
sudo useradd -r -m -s /bin/bash kafka
sudo usermod -aG wheel kafka

# Download and install Kafka
cd /tmp
wget https://downloads.apache.org/kafka/3.6.1/kafka_2.13-3.6.1.tgz
tar -xzf kafka_2.13-3.6.1.tgz

cd /tmp
wget https://archive.apache.org/dist/kafka/3.6.1/kafka_2.13-3.6.1.tgz
tar -xzf kafka_2.13-3.6.1.tgz


# Move to installation directory
sudo mv kafka_2.13-3.6.1 /opt/kafka
sudo chown -R kafka:kafka /opt/kafka

# Create Kafka data directories
sudo mkdir -p /var/lib/kafka-logs /var/lib/zookeeper
sudo chown -R kafka:kafka /var/lib/kafka-logs /var/lib/zookeeper

Configuration
# Configure Kafka server properties
sudo tee /opt/kafka/config/server.properties > /dev/null << 'EOF'
broker.id=0
listeners=PLAINTEXT://localhost:9092
log.dirs=/var/lib/kafka-logs
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
num.partitions=1
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=localhost:2181
zookeeper.connection.timeout.ms=18000
group.initial.rebalance.delay.ms=0
auto.create.topics.enable=true
delete.topic.enable=true
EOF

# Configure Zookeeper
sudo tee /opt/kafka/config/zookeeper.properties > /dev/null << 'EOF'
dataDir=/var/lib/zookeeper
clientPort=2181
maxClientCnxns=0
admin.enableServer=false
tickTime=2000
initLimit=10
syncLimit=5
EOF

# Create systemd service for Zookeeper
sudo tee /etc/systemd/system/zookeeper.service > /dev/null << 'EOF'
[Unit]
Description=Apache Zookeeper server (Kafka)
Documentation=http://zookeeper.apache.org
Requires=network.target remote-fs.target
After=network.target remote-fs.target

[Service]
Type=simple
User=kafka
Group=kafka
Environment=JAVA_HOME=/usr/lib/jvm/java-17-openjdk
ExecStart=/bin/bash /opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
ExecStop=/opt/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Create systemd service for Kafka
sudo tee /etc/systemd/system/kafka.service > /dev/null << 'EOF'
[Unit]
Description=Apache Kafka server (broker)
Documentation=http://kafka.apache.org/documentation.html
Requires=zookeeper.service
After=zookeeper.service

[Service]
Type=simple
User=kafka
Group=kafka
Environment=JAVA_HOME=/usr/lib/jvm/java-17-openjdk
ExecStart=/bin/bash  /opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Enable and start services
sudo systemctl daemon-reload
sudo systemctl enable zookeeper kafka
sudo systemctl start zookeeper
sleep 10
sudo systemctl start kafka

# Verify installation
sudo systemctl status zookeeper kafka

# Create test topics
sudo -u kafka /opt/kafka/bin/kafka-topics.sh --create --topic document-processing --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
sudo -u kafka /opt/kafka/bin/kafka-topics.sh --create --topic ai-responses --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1

Step 5: Install and Configure Redis
# Install Redis
sudo dnf install -y redis

# Configure Redis
sudo tee /etc/redis.conf > /dev/null << 'EOF'
bind 127.0.0.1 ::1
port 6379
timeout 0
keepalive 300
daemonize yes
supervised systemd
pidfile /var/run/redis/redis-server.pid
loglevel notice
logfile /var/log/redis/redis-server.log
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /var/lib/redis
maxmemory 256mb
maxmemory-policy allkeys-lru
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
EOF

# Enable and start Redis
sudo systemctl enable redis --now

# Test Redis
redis-cli ping  # Should return PONG

Step 6: Install and Configure MinIO Object Storage
# Download MinIO server binary
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/

# Create MinIO user and directories
sudo useradd -r -s /sbin/nologin -d /opt/minio minio
sudo mkdir -p /opt/minio /var/lib/minio
sudo chown minio:minio /opt/minio /var/lib/minio

# Create MinIO configuration
sudo tee /opt/minio/minio.conf > /dev/null << 'EOF'
MINIO_VOLUMES="/var/lib/minio"
MINIO_OPTS="--console-address :9001"
MINIO_ROOT_USER="minioadmin"
MINIO_ROOT_PASSWORD="MinioPassword123"
EOF

# Create systemd service for MinIO
sudo tee /etc/systemd/system/minio.service > /dev/null << 'EOF'
[Unit]
Description=MinIO High Performance Object Storage
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
User=minio
Group=minio
ProtectProc=invisible
EnvironmentFile=/opt/minio/minio.conf
ExecStartPre=/bin/bash -c 'if [ -z "${MINIO_VOLUMES}" ]; then echo "Variable MINIO_VOLUMES not set in /opt/minio/minio.conf"; exit 1; fi'
ExecStart=/usr/local/bin/minio server $MINIO_VOLUMES $MINIO_OPTS
Restart=always
LimitNOFILE=65536
TasksMax=infinity
TimeoutStopSec=infinity
SendSIGKILL=no

[Install]
WantedBy=multi-user.target
EOF

# Enable and start MinIO
sudo systemctl daemon-reload
sudo systemctl enable minio --now

# Verify MinIO is running
curl http://localhost:9000
curl http://localhost:9001  # Web console

Step 7: Install and Configure Ollama (Local LLM)
# Install Ollama using the official installer
curl -fsSL https://ollama.com/install.sh | sh

# Verify installation
ollama --version

# Create Ollama systemd service configuration
sudo tee /etc/systemd/system/ollama.service > /dev/null << 'EOF'
[Unit]
Description=Ollama AI Server
After=network.target

[Service]
Type=exec
User=ollama
Group=ollama
ExecStart=/usr/local/bin/ollama serve
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_MODELS=/opt/ai-system/models"
Restart=always
RestartSec=3
LimitNOFILE=65536

[Install]
WantedBy=default.target
EOF

# Create ollama user if not exists
sudo useradd -r -s /bin/false -m -d /usr/share/ollama ollama

# Set up models directory
sudo mkdir -p /opt/ai-system/models
sudo chown ollama:ollama /opt/ai-system/models

# Enable and start Ollama service
sudo systemctl daemon-reload
sudo systemctl enable ollama --now

# Wait for service to start
sleep 10

# Install recommended models
sudo -u ollama ollama pull llama3.1:8b      # General purpose model
sudo -u ollama ollama pull codellama:7b     # Code-focused model
sudo -u ollama ollama pull mistral:7b       # Lightweight efficient model
sudo -u ollama ollama pull nomic-embed-text # Embedding model

# List installed models
sudo -u ollama ollama list

# Test model interaction
sudo -u ollama ollama run llama3.1:8b "Hello, how are you?"

Step 8: Install Document Processing Tools
Install Unstructured.io
# Switch to ai-system user and activate virtual environment
sudo su - ai-system
cd /opt/ai-system
source venv/bin/activate

# Install Unstructured with all document support
pip install "unstructured[all-docs]"

# Install additional dependencies for better document processing
pip install "unstructured[pdf]" "unstructured[docx]" "unstructured[pptx]"

# Test Unstructured installation
python -c "from unstructured.partition.auto import partition; print('Unstructured installed successfully')"

Install Docling
# Install Docling with CPU-only support (for RHEL compatibility)
pip install docling --extra-index-url https://download.pytorch.org/whl/cpu

# Install additional OCR engines
pip install easyocr

# Configure Tesseract environment variable
echo 'export TESSDATA_PREFIX=/usr/share/tesseract/tessdata/' >> ~/.bashrc
source ~/.bashrc

# Test Docling installation
python -c "from docling.document_converter import DocumentConverter; print('Docling installed successfully')"

# Test with a simple document conversion
docling --help

Step 9: Install and Configure Nginx (Reverse Proxy)
# Exit ai-system user to become root
exit

# Create Nginx repository configuration
sudo tee /etc/yum.repos.d/nginx.repo > /dev/null << 'EOF'
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOF

# Install Nginx
sudo dnf install -y nginx

# Create main application configuration
sudo tee /etc/nginx/conf.d/ai-document-system.conf > /dev/null << 'EOF'
server {
    listen 80;
    server_name localhost;
    client_max_body_size 100M;

    # Main application (Streamlit)
    location / {
        proxy_pass http://localhost:8501;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 86400;
    }

    # FastAPI backend
    location /api/ {
        proxy_pass http://localhost:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 300;
        client_max_body_size 100M;
    }

    # MinIO S3 API
    location /storage/ {
        proxy_pass http://localhost:9000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        client_max_body_size 100M;
    }

    # MinIO Console
    location /minio-console/ {
        proxy_pass http://localhost:9001/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF

# Remove default configuration
sudo rm -f /etc/nginx/conf.d/default.conf

# Test configuration and start Nginx
sudo nginx -t
sudo systemctl enable nginx --now

# Configure firewall
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

Application Development and Deployment
Step 10: Create FastAPI Backend Application
# Switch to ai-system user
sudo su - ai-system
cd /opt/ai-system
source venv/bin/activate

# Create application structure
mkdir -p apps/{backend,frontend} config logs

# Create FastAPI backend
cat > apps/backend/main.py << 'EOF'
from fastapi import FastAPI, File, UploadFile, HTTPException, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
import asyncio
import json
import os
import tempfile
import logging
from datetime import datetime
from typing import Optional, List
import uvicorn

# Database and external services
import psycopg2
from kafka import KafkaProducer, KafkaConsumer
import redis
import requests
from minio import Minio
import ollama

# Document processing libraries
from unstructured.partition.auto import partition
from docling.document_converter import DocumentConverter
from docling.datamodel.base_models import ConversionStatus
from docling.datamodel.pipeline_options import PipelineOptions

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="AI Document Processing System", version="1.0.0")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Configuration
DATABASE_URL = "postgresql://ai_user:AiUserPassword123@localhost/ai_document_db"
REDIS_URL = "redis://localhost:6379"
KAFKA_BOOTSTRAP_SERVERS = ["localhost:9092"]
OLLAMA_HOST = "http://localhost:11434"
MINIO_ENDPOINT = "localhost:9000"
MINIO_ACCESS_KEY = "minioadmin"
MINIO_SECRET_KEY = "MinioPassword123"

# Initialize connections
def get_db_connection():
    return psycopg2.connect(DATABASE_URL)

redis_client = redis.Redis.from_url(REDIS_URL)

minio_client = Minio(
    MINIO_ENDPOINT,
    access_key=MINIO_ACCESS_KEY,
    secret_key=MINIO_SECRET_KEY,
    secure=False
)

kafka_producer = KafkaProducer(
    bootstrap_servers=KAFKA_BOOTSTRAP_SERVERS,
    value_serializer=lambda x: json.dumps(x).encode('utf-8')
)

# Initialize document converter
pipeline_options = PipelineOptions()
pipeline_options.do_ocr = True
doc_converter = DocumentConverter(pipeline_options=pipeline_options)

@app.on_startup
async def startup_event():
    """Initialize MinIO buckets and other startup tasks"""
    try:
        # Create MinIO buckets if they don't exist
        buckets = ["documents", "processed", "exports"]
        for bucket in buckets:
            if not minio_client.bucket_exists(bucket):
                minio_client.make_bucket(bucket)
                logger.info(f"Created MinIO bucket: {bucket}")
    except Exception as e:
        logger.error(f"Startup error: {str(e)}")

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    services = {
        "database": check_database(),
        "redis": check_redis(),
        "kafka": check_kafka(),
        "ollama": check_ollama(),
        "minio": check_minio()
    }
    
    return {
        "status": "healthy" if all(s == "healthy" for s in services.values()) else "unhealthy",
        "timestamp": datetime.now().isoformat(),
        "services": services
    }

def check_database():
    try:
        conn = get_db_connection()
        conn.close()
        return "healthy"
    except Exception as e:
        return f"error: {str(e)}"

def check_redis():
    try:
        redis_client.ping()
        return "healthy"
    except Exception as e:
        return f"error: {str(e)}"

def check_kafka():
    try:
        producer = KafkaProducer(bootstrap_servers=KAFKA_BOOTSTRAP_SERVERS)
        producer.close()
        return "healthy"
    except Exception as e:
        return f"error: {str(e)}"

def check_ollama():
    try:
        response = requests.get(f"{OLLAMA_HOST}/api/tags", timeout=5)
        return "healthy" if response.status_code == 200 else "error"
    except Exception as e:
        return f"error: {str(e)}"

def check_minio():
    try:
        minio_client.list_buckets()
        return "healthy"
    except Exception as e:
        return f"error: {str(e)}"

@app.post("/upload-document")
async def upload_document(
    background_tasks: BackgroundTasks,
    file: UploadFile = File(...)
):
    """Upload and queue document for processing"""
    try:
        # Read file content
        file_content = await file.read()
        object_name = f"documents/{datetime.now().strftime('%Y/%m/%d')}/{file.filename}"
        
        # Upload to MinIO
        minio_client.put_object(
            "documents",
            object_name,
            data=io.BytesIO(file_content),
            length=len(file_content),
            content_type=file.content_type or "application/octet-stream"
        )
        
        # Create processing message
        message = {
            "file_name": file.filename,
            "object_name": object_name,
            "content_type": file.content_type,
            "size": len(file_content),
            "timestamp": datetime.now().isoformat()
        }
        
        # Send to Kafka for processing
        kafka_producer.send('document-processing', message)
        
        # Cache metadata in Redis
        redis_client.setex(
            f"doc:{object_name}",
            3600,  # 1 hour expiry
            json.dumps(message)
        )
        
        # Add background processing task
        background_tasks.add_task(process_document_background, object_name)
        
        return {
            "message": "Document uploaded successfully",
            "object_name": object_name,
            "processing_status": "queued"
        }
        
    except Exception as e:
        logger.error(f"Error uploading document: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

async def process_document_background(object_name: str):
    """Background task to process document"""
    try:
        # Get file from MinIO
        response = minio_client.get_object("documents", object_name)
        file_content = response.read()
        
        # Create temporary file for processing
        with tempfile.NamedTemporaryFile(delete=False) as temp_file:
            temp_file.write(file_content)
            temp_file_path = temp_file.name
        
        try:
            # Process with Docling first (better for complex PDFs)
            try:
                result = doc_converter.convert(temp_file_path)
                extracted_text = result.document.export_to_markdown()
                processing_method = "docling"
            except Exception as docling_error:
                logger.warning(f"Docling failed, trying Unstructured: {str(docling_error)}")
                # Fallback to Unstructured
                elements = partition(filename=temp_file_path)
                extracted_text = "\n\n".join([str(el) for el in elements])
                processing_method = "unstructured"
            
            # Generate AI summary using Ollama
            try:
                summary_response = requests.post(
                    f"{OLLAMA_HOST}/api/generate",
                    json={
                        "model": "llama3.1:8b",
                        "prompt": f"Summarize and extract key information from this document:\n\n{extracted_text[:4000]}...",
                        "stream": False
                    },
                    timeout=60
                )
                
                if summary_response.status_code == 200:
                    ai_summary = summary_response.json()["response"]
                else:
                    ai_summary = "AI summary generation failed"
                    
            except Exception as ai_error:
                logger.error(f"AI summary generation failed: {str(ai_error)}")
                ai_summary = "AI summary not available"
            
            # Store results in database
            conn = get_db_connection()
            cursor = conn.cursor()
            cursor.execute("""
                INSERT INTO processed_documents 
                (object_name, extracted_text, ai_summary, metadata, processed_at)
                VALUES (%s, %s, %s, %s, %s)
                ON CONFLICT (object_name) DO UPDATE SET
                extracted_text = EXCLUDED.extracted_text,
                ai_summary = EXCLUDED.ai_summary,
                metadata = EXCLUDED.metadata,
                processed_at = EXCLUDED.processed_at
            """, (
                object_name, 
                extracted_text, 
                ai_summary,
                json.dumps({"processing_method": processing_method}),
                datetime.now()
            ))
            conn.commit()
            conn.close()
            
            # Cache result in Redis
            result = {
                "object_name": object_name,
                "extracted_text": extracted_text,
                "ai_summary": ai_summary,
                "processing_method": processing_method,
                "processed_at": datetime.now().isoformat()
            }
            
            redis_client.setex(
                f"processed:{object_name}",
                7200,  # 2 hours expiry
                json.dumps(result)
            )
            
            logger.info(f"Successfully processed document: {object_name}")
            
        finally:
            # Clean up temporary file
            os.unlink(temp_file_path)
            
    except Exception as e:
        logger.error(f"Error processing document {object_name}: {str(e)}")

@app.get("/document-status/{object_name:path}")
async def get_document_status(object_name: str):
    """Get processing status of a document"""
    try:
        # Check if processed (in Redis first for speed)
        cached_result = redis_client.get(f"processed:{object_name}")
        if cached_result:
            return {
                "status": "completed",
                "result": json.loads(cached_result)
            }
        
        # Check database
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute(
            "SELECT extracted_text, ai_summary, metadata, processed_at FROM processed_documents WHERE object_name = %s",
            (object_name,)
        )
        result = cursor.fetchone()
        conn.close()
        
        if result:
            return {
                "status": "completed",
                "result": {
                    "object_name": object_name,
                    "extracted_text": result[^0],
                    "ai_summary": result[^1],
                    "metadata": result[^2],
                    "processed_at": result[^3].isoformat() if result[^3] else None
                }
            }
        
        # Check if queued
        queued = redis_client.get(f"doc:{object_name}")
        if queued:
            return {"status": "processing"}
        
        return {"status": "not_found"}
        
    except Exception as e:
        logger.error(f"Error checking document status: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/chat")
async def chat_with_ai(query: str, context_documents: Optional[List[str]] = None):
    """Chat with AI using Ollama with optional document context"""
    try:
        # Build context from documents if provided
        context = ""
        if context_documents:
            conn = get_db_connection()
            cursor = conn.cursor()
            
            for doc_name in context_documents:
                cursor.execute(
                    "SELECT extracted_text FROM processed_documents WHERE object_name = %s",
                    (doc_name,)
                )
                result = cursor.fetchone()
                if result:
                    context += f"\n\nDocument: {doc_name}\n{result[^0][:2000]}..."
            
            conn.close()
        
        # Prepare prompt with context
        if context:
            full_prompt = f"Context from documents:\n{context}\n\nUser question: {query}\n\nPlease answer based on the provided context:"
        else:
            full_prompt = query
        
        # Generate response using Ollama
        response = requests.post(
            f"{OLLAMA_HOST}/api/generate",
            json={
                "model": "llama3.1:8b",
                "prompt": full_prompt,
                "stream": False
            },
            timeout=60
        )
        
        if response.status_code == 200:
            ai_response = response.json()["response"]
            
            # Store chat session
            conn = get_db_connection()
            cursor = conn.cursor()
            cursor.execute("""
                INSERT INTO chat_sessions (user_query, ai_response, model_used)
                VALUES (%s, %s, %s)
            """, (query, ai_response, "llama3.1:8b"))
            conn.commit()
            conn.close()
            
            return {
                "query": query,
                "response": ai_response,
                "model": "llama3.1:8b",
                "context_used": bool(context),
                "timestamp": datetime.now().isoformat()
            }
        else:
            raise HTTPException(status_code=500, detail="AI service unavailable")
        
    except Exception as e:
        logger.error(f"Error in AI chat: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/documents")
async def list_documents():
    """List all processed documents"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("""
            SELECT object_name, original_filename, content_type, 
                   LENGTH(extracted_text) as text_length,
                   processed_at, created_at 
            FROM processed_documents 
            ORDER BY processed_at DESC 
            LIMIT 100
        """)
        
        documents = []
        for row in cursor.fetchall():
            documents.append({
                "object_name": row[^0],
                "original_filename": row[^1],
                "content_type": row[^2],
                "text_length": row[^3],
                "processed_at": row[^4].isoformat() if row[^4] else None,
                "created_at": row[^5].isoformat() if row[^5] else None
            })
        
        conn.close()
        return {"documents": documents}
        
    except Exception as e:
        logger.error(f"Error listing documents: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
EOF




cat > apps/backend/main.py << 'EOF'
cat > /opt/ai-system/apps/backend/main.py << 'EOF'
from fastapi import FastAPI, File, UploadFile, HTTPException, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
import asyncio
import json
import os
import tempfile
import io
import logging
from datetime import datetime
from typing import Optional, List
import uvicorn

# Database and external services
import psycopg2
from kafka import KafkaProducer, KafkaConsumer
import redis
import requests
from minio import Minio
import ollama

# Document processing libraries
from unstructured.partition.auto import partition
from docling.document_converter import DocumentConverter
from docling.datamodel.base_models import ConversionStatus
from docling.datamodel.pipeline_options import PipelineOptions

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="AI Document Processing System", version="1.0.0")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Configuration
DATABASE_URL = "postgresql://ai_user:AiUserPassword123@localhost/ai_document_db"
REDIS_URL = "redis://localhost:6379"
KAFKA_BOOTSTRAP_SERVERS = ["localhost:9092"]
OLLAMA_HOST = "http://localhost:11434"
MINIO_ENDPOINT = "localhost:9000"
MINIO_ACCESS_KEY = "minioadmin"
MINIO_SECRET_KEY = "MinioPassword123"

# Initialize connections
def get_db_connection():
    return psycopg2.connect(DATABASE_URL)

redis_client = redis.Redis.from_url(REDIS_URL)

minio_client = Minio(
    MINIO_ENDPOINT,
    access_key=MINIO_ACCESS_KEY,
    secret_key=MINIO_SECRET_KEY,
    secure=False
)

kafka_producer = KafkaProducer(
    bootstrap_servers=KAFKA_BOOTSTRAP_SERVERS,
    value_serializer=lambda x: json.dumps(x).encode('utf-8')
)

# Initialize document converter
pipeline_options = PipelineOptions()
pipeline_options.do_ocr = True
doc_converter = DocumentConverter(pipeline_options=pipeline_options)

@app.on_event("startup")
async def startup_event():
    """Initialize MinIO buckets and other startup tasks"""
    try:
        # Create MinIO buckets if they don't exist
        buckets = ["documents", "processed", "exports"]
        for bucket in buckets:
            if not minio_client.bucket_exists(bucket):
                minio_client.make_bucket(bucket)
                logger.info(f"Created MinIO bucket: {bucket}")
    except Exception as e:
        logger.error(f"Startup error: {str(e)}")

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    services = {
        "database": check_database(),
        "redis": check_redis(),
        "kafka": check_kafka(),
        "ollama": check_ollama(),
        "minio": check_minio()
    }

    return {
        "status": "healthy" if all(s == "healthy" for s in services.values()) else "unhealthy",
        "timestamp": datetime.now().isoformat(),
        "services": services
    }

def check_database():
    try:
        conn = get_db_connection()
        conn.close()
        return "healthy"
    except Exception as e:
        return f"error: {str(e)}"

def check_redis():
    try:
        redis_client.ping()
        return "healthy"
    except Exception as e:
        return f"error: {str(e)}"

def check_kafka():
    try:
        producer = KafkaProducer(bootstrap_servers=KAFKA_BOOTSTRAP_SERVERS)
        producer.close()
        return "healthy"
    except Exception as e:
        return f"error: {str(e)}"

def check_ollama():
    try:
        response = requests.get(f"{OLLAMA_HOST}/api/tags", timeout=5)
        return "healthy" if response.status_code == 200 else "error"
    except Exception as e:
        return f"error: {str(e)}"

def check_minio():
    try:
        minio_client.list_buckets()
        return "healthy"
    except Exception as e:
        return f"error: {str(e)}"

@app.post("/upload-document")
async def upload_document(
    background_tasks: BackgroundTasks,
    file: UploadFile = File(...)
):
    """Upload and queue document for processing"""
    try:
        # Read file content
        file_content = await file.read()
        object_name = f"documents/{datetime.now().strftime('%Y/%m/%d')}/{file.filename}"

        # Upload to MinIO
        minio_client.put_object(
            "documents",
            object_name,
            data=io.BytesIO(file_content),
            length=len(file_content),
            content_type=file.content_type or "application/octet-stream"
        )

        # Create processing message
        message = {
            "file_name": file.filename,
            "object_name": object_name,
            "content_type": file.content_type,
            "size": len(file_content),
            "timestamp": datetime.now().isoformat()
        }

        # Send to Kafka for processing
        kafka_producer.send('document-processing', message)

        # Cache metadata in Redis
        redis_client.setex(
            f"doc:{object_name}",
            3600, # 1 hour expiry
            json.dumps(message)
        )

        # Add background processing task
        background_tasks.add_task(process_document_background, object_name)

        return {
            "message": "Document uploaded successfully",
            "object_name": object_name,
            "processing_status": "queued"
        }

    except Exception as e:
        logger.error(f"Error uploading document: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

async def process_document_background(object_name: str):
    """Background task to process document"""
    try:
        # Get file from MinIO
        response = minio_client.get_object("documents", object_name)
        file_content = response.read()

        # Create temporary file for processing
        with tempfile.NamedTemporaryFile(delete=False) as temp_file:
            temp_file.write(file_content)
            temp_file_path = temp_file.name

        try:
            # Process with Docling first (better for complex PDFs)
            try:
                result = doc_converter.convert(temp_file_path)
                extracted_text = result.document.export_to_markdown()
                processing_method = "docling"
            except Exception as docling_error:
                logger.warning(f"Docling failed, trying Unstructured: {str(docling_error)}")
                # Fallback to Unstructured
                elements = partition(filename=temp_file_path)
                extracted_text = "\n\n".join([str(el) for el in elements])
                processing_method = "unstructured"

            # Generate AI summary using Ollama
            try:
                summary_response = requests.post(
                    f"{OLLAMA_HOST}/api/generate",
                    json={
                        "model": "llama3.1:8b",
                        "prompt": f"Summarize and extract key information from this document:\n\n{extracted_text[:4000]}...",
                        "stream": False
                    },
                    timeout=60
                )

                if summary_response.status_code == 200:
                    ai_summary = summary_response.json()["response"]
                else:
                    ai_summary = "AI summary generation failed"

            except Exception as ai_error:
                logger.error(f"AI summary generation failed: {str(ai_error)}")
                ai_summary = "AI summary not available"

            # Store results in database
            conn = get_db_connection()
            cursor = conn.cursor()
            cursor.execute("""
                INSERT INTO processed_documents
                (object_name, extracted_text, ai_summary, metadata, processed_at)
                VALUES (%s, %s, %s, %s, %s)
                ON CONFLICT (object_name) DO UPDATE SET
                extracted_text = EXCLUDED.extracted_text,
                ai_summary = EXCLUDED.ai_summary,
                metadata = EXCLUDED.metadata,
                processed_at = EXCLUDED.processed_at
            """, (
                object_name,
                extracted_text,
                ai_summary,
                json.dumps({"processing_method": processing_method}),
                datetime.now()
            ))
            conn.commit()
            conn.close()

            # Cache result in Redis
            result = {
                "object_name": object_name,
                "extracted_text": extracted_text,
                "ai_summary": ai_summary,
                "processing_method": processing_method,
                "processed_at": datetime.now().isoformat()
            }

            redis_client.setex(
                f"processed:{object_name}",
                7200, # 2 hours expiry
                json.dumps(result)
            )

            logger.info(f"Successfully processed document: {object_name}")

        finally:
            # Clean up temporary file
            os.unlink(temp_file_path)

    except Exception as e:
        logger.error(f"Error processing document {object_name}: {str(e)}")

@app.get("/document-status/{object_name:path}")
async def get_document_status(object_name: str):
    """Get processing status of a document"""
    try:
        # Check if processed (in Redis first for speed)
        cached_result = redis_client.get(f"processed:{object_name}")
        if cached_result:
            return {
                "status": "completed",
                "result": json.loads(cached_result)
            }

        # Check database
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute(
            "SELECT extracted_text, ai_summary, metadata, processed_at FROM processed_documents WHERE object_name = %s",
            (object_name,)
        )
        result = cursor.fetchone()
        conn.close()

        if result:
            return {
                "status": "completed",
                "result": {
                    "object_name": object_name,
                    "extracted_text": result[0],
                    "ai_summary": result[1],
                    "metadata": result[2],
                    "processed_at": result[3].isoformat() if result[3] else None
                }
            }

        # Check if queued
        queued = redis_client.get(f"doc:{object_name}")
        if queued:
            return {"status": "processing"}

        return {"status": "not_found"}

    except Exception as e:
        logger.error(f"Error checking document status: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/chat")
async def chat_with_ai(query: str, context_documents: Optional[List[str]] = None):
    """Chat with AI using Ollama with optional document context"""
    try:
        # Build context from documents if provided
        context = ""
        if context_documents:
            conn = get_db_connection()
            cursor = conn.cursor()

            for doc_name in context_documents:
                cursor.execute(
                    "SELECT extracted_text FROM processed_documents WHERE object_name = %s",
                    (doc_name,)
                )
                result = cursor.fetchone()
                if result:
                    context += f"\n\nDocument: {doc_name}\n{result[0][:2000]}..."

            conn.close()

        # Prepare prompt with context
        if context:
            full_prompt = f"Context from documents:\n{context}\n\nUser question: {query}\n\nPlease answer based on the provided context:"
        else:
            full_prompt = query

        # Generate response using Ollama
        response = requests.post(
            f"{OLLAMA_HOST}/api/generate",
            json={
                "model": "llama3.1:8b",
                "prompt": full_prompt,
                "stream": False
            },
            timeout=60
        )

        if response.status_code == 200:
            ai_response = response.json()["response"]

            # Store chat session
            conn = get_db_connection()
            cursor = conn.cursor()
            cursor.execute("""
                INSERT INTO chat_sessions (user_query, ai_response, model_used)
                VALUES (%s, %s, %s)
            """, (query, ai_response, "llama3.1:8b"))
            conn.commit()
            conn.close()

            return {
                "query": query,
                "response": ai_response,
                "model": "llama3.1:8b",
                "context_used": bool(context),
                "timestamp": datetime.now().isoformat()
            }
        else:
            raise HTTPException(status_code=500, detail="AI service unavailable")

    except Exception as e:
        logger.error(f"Error in AI chat: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/documents")
async def list_documents():
    """List all processed documents"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("""
            SELECT object_name, original_filename, content_type,
                   LENGTH(extracted_text) as text_length,
                   processed_at, created_at
            FROM processed_documents
            ORDER BY processed_at DESC
            LIMIT 100
        """)

        documents = []
        for row in cursor.fetchall():
            documents.append({
                "object_name": row[0],
                "original_filename": row[1],
                "content_type": row[2],
                "text_length": row[3],
                "processed_at": row[4].isoformat() if row[4] else None,
                "created_at": row[5].isoformat() if row[5] else None
            })

        conn.close()
        return {"documents": documents}

    except Exception as e:
        logger.error(f"Error listing documents: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
EOF


# Add missing imports
sed -i '1i import io' apps/backend/main.py

#Optional
 sudo nano /opt/ai-system/apps/backend/main.py

import io
from fastapi import FastAPI, File, UploadFile, HTTPException, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
import asyncio
import json
import os
import tempfile
import logging
from datetime import datetime
from typing import Optional, List
import uvicorn

# Database and external services
import psycopg2
from kafka import KafkaProducer, KafkaConsumer
import redis
import requests
from minio import Minio
import ollama

# Document processing libraries
from unstructured.partition.auto import partition
from docling.document_converter import DocumentConverter, PdfFormatOption
from docling.datamodel.base_models import ConversionStatus
from docling.datamodel.pipeline_options import PdfPipelineOptions

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="AI Document Processing System", version="1.0.0")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Configuration
DATABASE_URL = "postgresql://ai_user:AiUserPassword123@localhost/ai_document_db"
REDIS_URL = "redis://localhost:6379"
KAFKA_BOOTSTRAP_SERVERS = ["localhost:9092"]
OLLAMA_HOST = "http://localhost:11434"
MINIO_ENDPOINT = "localhost:9000"
MINIO_ACCESS_KEY = "minioadmin"
MINIO_SECRET_KEY = "MinioPassword123"

# Initialize connections
def get_db_connection():
    return psycopg2.connect(DATABASE_URL)

redis_client = redis.Redis.from_url(REDIS_URL)

minio_client = Minio(
    MINIO_ENDPOINT,
    access_key=MINIO_ACCESS_KEY,
    secret_key=MINIO_SECRET_KEY,
    secure=False
)

kafka_producer = KafkaProducer(
    bootstrap_servers=KAFKA_BOOTSTRAP_SERVERS,
    value_serializer=lambda x: json.dumps(x).encode('utf-8')
)

# Initialize document converter (FIXED)
pipeline_options = PdfPipelineOptions()
pipeline_options.do_ocr = True

format_options = {
    "pdf": PdfFormatOption(pipeline_options=pipeline_options)
    # Add DOCX, PPTX, etc. if needed
}

doc_converter = DocumentConverter(format_options=format_options)

@app.on_event("startup")
async def startup_event():
    """Initialize MinIO buckets and other startup tasks"""
    try:
        buckets = ["documents", "processed", "exports"]
        for bucket in buckets:
            if not minio_client.bucket_exists(bucket):
                minio_client.make_bucket(bucket)
                logger.info(f"Created MinIO bucket: {bucket}")
    except Exception as e:
        logger.error(f"Startup error: {str(e)}")

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    services = {
        "database": check_database(),
        "redis": check_redis(),
        "kafka": check_kafka(),
        "ollama": check_ollama(),
        "minio": check_minio()
    }

    return {
        "status": "healthy" if all(s == "healthy" for s in services.values()) else "unhealthy",
        "timestamp": datetime.now().isoformat(),
        "services": services
    }

def check_database():
    try:
        conn = get_db_connection()
        conn.close()
        return "healthy"
    except Exception as e:
        return f"error: {str(e)}"

def check_redis():
    try:
        redis_client.ping()
        return "healthy"
    except Exception as e:
        return f"error: {str(e)}"

def check_kafka():
    try:
        producer = KafkaProducer(bootstrap_servers=KAFKA_BOOTSTRAP_SERVERS)
        producer.close()
        return "healthy"
    except Exception as e:
        return f"error: {str(e)}"

def check_ollama():
    try:
        response = requests.get(f"{OLLAMA_HOST}/api/tags", timeout=5)
        return "healthy" if response.status_code == 200 else "error"
    except Exception as e:
        return f"error: {str(e)}"

def check_minio():
    try:
        minio_client.list_buckets()
        return "healthy"
    except Exception as e:
        return f"error: {str(e)}"

@app.post("/upload-document")
async def upload_document(
    background_tasks: BackgroundTasks,
    file: UploadFile = File(...)
):
    """Upload and queue document for processing"""
    try:
        file_content = await file.read()
        object_name = f"documents/{datetime.now().strftime('%Y/%m/%d')}/{file.filename}"
        minio_client.put_object(
            "documents",
            object_name,
            data=io.BytesIO(file_content),
            length=len(file_content),
            content_type=file.content_type or "application/octet-stream"
        )

        message = {
            "file_name": file.filename,
            "object_name": object_name,
            "content_type": file.content_type,
            "size": len(file_content),
            "timestamp": datetime.now().isoformat()
        }

        kafka_producer.send('document-processing', message)

        redis_client.setex(
            f"doc:{object_name}",
            3600,
            json.dumps(message)
        )

        background_tasks.add_task(process_document_background, object_name)

        return {
            "message": "Document uploaded successfully",
            "object_name": object_name,
            "processing_status": "queued"
        }

    except Exception as e:
        logger.error(f"Error uploading document: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

async def process_document_background(object_name: str):
    """Background task to process document"""
    try:
        response = minio_client.get_object("documents", object_name)
        file_content = response.read()

        with tempfile.NamedTemporaryFile(delete=False) as temp_file:
            temp_file.write(file_content)
            temp_file_path = temp_file.name

        try:
            try:
                result = doc_converter.convert(temp_file_path)
                extracted_text = result.document.export_to_markdown()
                processing_method = "docling"
            except Exception as docling_error:
                logger.warning(f"Docling failed, trying Unstructured: {str(docling_error)}")
                elements = partition(filename=temp_file_path)
                extracted_text = "\n\n".join([str(el) for el in elements])
                processing_method = "unstructured"

            try:
                summary_response = requests.post(
                    f"{OLLAMA_HOST}/api/generate",
                    json={
                        "model": "llama3.1:8b",
                        "prompt": f"Summarize and extract key information from this document:\n\n{extracted_text[:4000]}...",
                        "stream": False
                    },
                    timeout=60
                )

                if summary_response.status_code == 200:
                    ai_summary = summary_response.json()["response"]
                else:
                    ai_summary = "AI summary generation failed"

            except Exception as ai_error:
                logger.error(f"AI summary generation failed: {str(ai_error)}")
                ai_summary = "AI summary not available"

            conn = get_db_connection()
            cursor = conn.cursor()
            cursor.execute("""
                INSERT INTO processed_documents
                (object_name, extracted_text, ai_summary, metadata, processed_at)
                VALUES (%s, %s, %s, %s, %s)
                ON CONFLICT (object_name) DO UPDATE SET
                extracted_text = EXCLUDED.extracted_text,
                ai_summary = EXCLUDED.ai_summary,
                metadata = EXCLUDED.metadata,
                processed_at = EXCLUDED.processed_at
            """, (
                object_name,
                extracted_text,
                ai_summary,
                json.dumps({"processing_method": processing_method}),
                datetime.now()
            ))
            conn.commit()
            conn.close()

            result = {
                "object_name": object_name,
                "extracted_text": extracted_text,
                "ai_summary": ai_summary,
                "processing_method": processing_method,
                "processed_at": datetime.now().isoformat()
            }

            redis_client.setex(
                f"processed:{object_name}",
                7200,
                json.dumps(result)
            )

            logger.info(f"Successfully processed document: {object_name}")

        finally:
            os.unlink(temp_file_path)

    except Exception as e:
        logger.error(f"Error processing document {object_name}: {str(e)}")

@app.get("/document-status/{object_name:path}")
async def get_document_status(object_name: str):
    """Get processing status of a document"""
    try:
        cached_result = redis_client.get(f"processed:{object_name}")
        if cached_result:
            return {
                "status": "completed",
                "result": json.loads(cached_result)
            }

        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute(
            "SELECT extracted_text, ai_summary, metadata, processed_at FROM processed_documents WHERE object_name = %s",
            (object_name,)
        )
        result = cursor.fetchone()
        conn.close()

        if result:
            return {
                "status": "completed",
                "result": {
                    "object_name": object_name,
                    "extracted_text": result[0],
                    "ai_summary": result[1],
                    "metadata": result[2],
                    "processed_at": result[3].isoformat() if result[3] else None
                }
            }

        queued = redis_client.get(f"doc:{object_name}")
        if queued:
            return {"status": "processing"}

        return {"status": "not_found"}

    except Exception as e:
        logger.error(f"Error checking document status: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/chat")
async def chat_with_ai(query: str, context_documents: Optional[List[str]] = None):
    """Chat with AI using Ollama with optional document context"""
    try:
        context = ""
        if context_documents:
            conn = get_db_connection()
            cursor = conn.cursor()

            for doc_name in context_documents:
                cursor.execute(
                    "SELECT extracted_text FROM processed_documents WHERE object_name = %s",
                    (doc_name,)
                )
                result = cursor.fetchone()
                if result:
                    context += f"\n\nDocument: {doc_name}\n{result[0][:2000]}..."
            conn.close()

        if context:
            full_prompt = f"Context from documents:\n{context}\n\nUser question: {query}\n\nPlease answer based on the provided context:"
        else:
            full_prompt = query

        response = requests.post(
            f"{OLLAMA_HOST}/api/generate",
            json={
                "model": "llama3.1:8b",
                "prompt": full_prompt,
                "stream": False
            },
            timeout=60
        )

        if response.status_code == 200:
            ai_response = response.json()["response"]

            conn = get_db_connection()
            cursor = conn.cursor()
            cursor.execute("""
                INSERT INTO chat_sessions (user_query, ai_response, model_used)
                VALUES (%s, %s, %s)
            """, (query, ai_response, "llama3.1:8b"))
            conn.commit()
            conn.close()

            return {
                "query": query,
                "response": ai_response,
                "model": "llama3.1:8b",
                "context_used": bool(context),
                "timestamp": datetime.now().isoformat()
            }
        else:
            raise HTTPException(status_code=500, detail="AI service unavailable")

    except Exception as e:
        logger.error(f"Error in AI chat: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/documents")
async def list_documents():
    """List all processed documents"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("""
            SELECT object_name, original_filename, content_type, 
                   LENGTH(extracted_text) as text_length,
                   processed_at, created_at 
            FROM processed_documents 
            ORDER BY processed_at DESC 
            LIMIT 100
        """)

        documents = []
        for row in cursor.fetchall():
            documents.append({
                "object_name": row[0],
                "original_filename": row[1],
                "content_type": row[2],
                "text_length": row[3],
                "processed_at": row[4].isoformat() if row[4] else None,
                "created_at": row[5].isoformat() if row[5] else None
            })
        conn.close()
        return {"documents": documents}

    except Exception as e:
        logger.error(f"Error listing documents: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)

Step 11: Create Streamlit Frontend Application
# Create Streamlit frontend
cat > apps/frontend/main.py << 'EOF'
import streamlit as st
import requests
import json
import pandas as pd
from datetime import datetime
import time

# Configure Streamlit page
st.set_page_config(
    page_title="AI Document Processing System",
    page_icon="📄",
    layout="wide",
    initial_sidebar_state="expanded"
)

# API base URL
API_BASE_URL = "http://localhost:8000"

def main():
    st.title("🤖 AI Document Processing System")
    st.markdown("Upload documents, extract content, and interact with AI on RHEL")
    
    # Sidebar navigation
    st.sidebar.title("Navigation")
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Home", "Document Upload", "Document Processing", "AI Chat", "Document Library", "System Health"]
    )
    
    if page == "Home":
        show_home()
    elif page == "Document Upload":
        show_document_upload()
    elif page == "Document Processing":
        show_document_processing()
    elif page == "AI Chat":
        show_ai_chat()
    elif page == "Document Library":
        show_document_library()
    elif page == "System Health":
        show_system_health()

def show_home():
    st.header("Welcome to AI Document Processing System")
    
    col1, col2, col3 = st.columns(3)
    
    with col1:
        st.subheader("📄 Document Processing")
        st.write("Upload and extract content from various document formats using Unstructured.io and Docling")
        st.write("- PDF documents with OCR")
        st.write("- Microsoft Office files")
        st.write("- Images with text extraction")
        
    with col2:
        st.subheader("🤖 AI Analysis")
        st.write("Get AI-powered summaries and insights using local Ollama models")
        st.write("- Llama 3.1 for general analysis")
        st.write("- CodeLlama for technical documents")
        st.write("- Mistral for efficient processing")
        
    with col3:
        st.subheader("💬 Interactive Chat")
        st.write("Chat with AI about your documents and get instant answers")
        st.write("- Context-aware responses")
        st.write("- Document-based Q&A")
        st.write("- Multi-document analysis")
    
    # Show recent activity
    st.header("System Overview")
    
    try:
        response = requests.get(f"{API_BASE_URL}/documents")
        if response.status_code == 200:
            docs = response.json()["documents"]
            
            col1, col2, col3, col4 = st.columns(4)
            
            with col1:
                st.metric("Documents Processed", len(docs))
            with col2:
                total_size = sum(doc.get("text_length", 0) for doc in docs)
                st.metric("Total Text Extracted", f"{total_size:,} chars")
            with col3:
                recent_docs = [doc for doc in docs if doc.get("processed_at")]
                st.metric("Recently Processed", len(recent_docs))
            with col4:
                st.metric("System Status", "🟢 Healthy")
                
            # Show recent documents
            if docs:
                st.subheader("Recent Documents")
                df = pd.DataFrame(docs[:5])
                if not df.empty:
                    st.dataframe(df, use_container_width=True)
        
    except Exception as e:
        st.error(f"Error loading system overview: {str(e)}")

def show_document_upload():
    st.header("📄 Document Upload")
    
    uploaded_file = st.file_uploader(
        "Choose a document to upload",
        type=['pdf', 'docx', 'doc', 'txt', 'html', 'xml', 'csv', 'xlsx', 'pptx', 'jpg', 'jpeg', 'png'],
        help="Supported formats: PDF, DOCX, TXT, HTML, XML, CSV, XLSX, PPTX, Images"
    )
    
    if uploaded_file is not None:
        st.write("File details:")
        st.write(f"- **Name:** {uploaded_file.name}")
        st.write(f"- **Type:** {uploaded_file.type}")
        st.write(f"- **Size:** {uploaded_file.size:,} bytes")
        
        if st.button("Upload Document", type="primary"):
            with st.spinner("Uploading document..."):
                try:
                    files = {"file": (uploaded_file.name, uploaded_file.getvalue(), uploaded_file.type)}
                    response = requests.post(f"{API_BASE_URL}/upload-document", files=files)
                    
                    if response.status_code == 200:
                        result = response.json()
                        st.success("Document uploaded successfully!")
                        st.json(result)
                        
                        # Store object name in session state for processing
                        st.session_state.last_uploaded = result['object_name']
                        
                        # Show processing status
                        st.info("Processing started automatically. Check the 'Document Processing' page for status updates.")
                        
                    else:
                        st.error(f"Upload failed: {response.text}")
                        
                except Exception as e:
                    st.error(f"Error uploading document: {str(e)}")

def show_document_processing():
    st.header("⚙️ Document Processing")
    
    # Check if there's a recently uploaded document
    if 'last_uploaded' in st.session_state:
        st.info(f"Last uploaded document: {st.session_state.last_uploaded}")
        
        col1, col2 = st.columns([1, 1])
        with col1:
            if st.button("Check Processing Status", type="primary"):
                check_processing_status(st.session_state.last_uploaded)
        
        with col2:
            if st.button("View Results"):
                show_document_results(st.session_state.last_uploaded)
    
    # Manual processing status check
    st.subheader("Manual Status Check")
    object_name = st.text_input("Enter document object name:")
    
    col1, col2 = st.columns([1, 1])
    with col1:
        if st.button("Check Status") and object_name:
            check_processing_status(object_name)
    
    with col2:
        if st.button("View Results") and object_name:
            show_document_results(object_name)

def check_processing_status(object_name):
    """Check and display processing status"""
    with st.spinner("Checking processing status..."):
        try:
            response = requests.get(f"{API_BASE_URL}/document-status/{object_name}")
            
            if response.status_code == 200:
                result = response.json()
                status = result["status"]
                
                if status == "completed":
                    st.success("✅ Document processing completed!")
                    st.json(result["result"])
                elif status == "processing":
                    st.warning("⏳ Document is being processed...")
                    st.info("Please wait and check again in a few moments.")
                elif status == "not_found":
                    st.error("❌ Document not found")
                else:
                    st.info(f"Status: {status}")
            else:
                st.error(f"Error checking status: {response.text}")
                
        except Exception as e:
            st.error(f"Error checking processing status: {str(e)}")

def show_document_results(object_name):
    """Show detailed document processing results"""
    try:
        response = requests.get(f"{API_BASE_URL}/document-status/{object_name}")
        
        if response.status_code == 200:
            result = response.json()
            
            if result["status"] == "completed":
                doc_result = result["result"]
                
                st.subheader("Document Processing Results")
                
                # Processing metadata
                col1, col2 = st.columns(2)
                with col1:
                    st.write(f"**Document:** {object_name}")
                    st.write(f"**Processed at:** {doc_result.get('processed_at', 'N/A')}")
                
                with col2:
                    metadata = doc_result.get('metadata', {})
                    if isinstance(metadata, str):
                        metadata = json.loads(metadata)
                    st.write(f"**Processing method:** {metadata.get('processing_method', 'N/A')}")
                
                # AI Summary
                st.subheader("🤖 AI Summary")
                if doc_result.get('ai_summary'):
                    st.write(doc_result['ai_summary'])
                else:
                    st.info("AI summary not available")
                
                # Extracted Text
                st.subheader("📝 Extracted Text")
                if doc_result.get('extracted_text'):
                    with st.expander("View extracted text", expanded=False):
                        st.text_area(
                            "Content", 
                            doc_result['extracted_text'], 
                            height=300,
                            disabled=True
                        )
                        
                        # Text statistics
                        text = doc_result['extracted_text']
                        st.write(f"**Statistics:**")
                        st.write(f"- Characters: {len(text):,}")
                        st.write(f"- Words: {len(text.split()):,}")
                        st.write(f"- Lines: {text.count(chr(10)) + 1:,}")
                else:
                    st.info("No text extracted")
            else:
                st.warning(f"Document status: {result['status']}")
        else:
            st.error(f"Error retrieving results: {response.text}")
            
    except Exception as e:
        st.error(f"Error showing document results: {str(e)}")

def show_ai_chat():
    st.header("💬 AI Chat")
    
    # Initialize chat history
    if "messages" not in st.session_state:
        st.session_state.messages = []
    
    # Document context selection
    st.sidebar.subheader("Document Context")
    try:
        response = requests.get(f"{API_BASE_URL}/documents")
        if response.status_code == 200:
            docs = response.json()["documents"]
            if docs:
                selected_docs = st.sidebar.multiselect(
                    "Select documents for context:",
                    options=[doc["object_name"] for doc in docs],
                    format_func=lambda x: x.split("/")[-1]  # Show only filename
                )
            else:
                selected_docs = []
                st.sidebar.info("No documents available for context")
        else:
            selected_docs = []
            st.sidebar.error("Error loading documents")
    except Exception as e:
        selected_docs = []
        st.sidebar.error(f"Error: {str(e)}")
    
    # Display chat messages
    for message in st.session_state.messages:
        with st.chat_message(message["role"]):
            st.markdown(message["content"])
    
    # Chat input
    if prompt := st.chat_input("What would you like to know?"):
        # Add user message to chat history
        st.session_state.messages.append({"role": "user", "content": prompt})
        with st.chat_message("user"):
            st.markdown(prompt)
        
        # Get AI response
        with st.chat_message("assistant"):
            with st.spinner("Thinking..."):
                try:
                    # Prepare request data
                    request_data = {"query": prompt}
                    if selected_docs:
                        request_data["context_documents"] = selected_docs
                    
                    response = requests.post(
                        f"{API_BASE_URL}/chat",
                        params=request_data
                    )
                    
                    if response.status_code == 200:
                        result = response.json()
                        ai_response = result['response']
                        
                        # Show context info if used
                        if result.get('context_used'):
                            st.info(f"Response based on {len(selected_docs)} document(s)")
                        
                        st.markdown(ai_response)
                        
                        # Add assistant response to chat history
                        st.session_state.messages.append({"role": "assistant", "content": ai_response})
                    else:
                        error_msg = f"Error: {response.text}"
                        st.error(error_msg)
                        st.session_state.messages.append({"role": "assistant", "content": error_msg})
                        
                except Exception as e:
                    error_msg = f"Error communicating with AI: {str(e)}"
                    st.error(error_msg)
                    st.session_state.messages.append({"role": "assistant", "content": error_msg})

def show_document_library():
    st.header("📚 Document Library")
    
    try:
        response = requests.get(f"{API_BASE_URL}/documents")
        
        if response.status_code == 200:
            docs = response.json()["documents"]
            
            if docs:
                st.write(f"Found {len(docs)} processed documents")
                
                # Create DataFrame for better display
                df = pd.DataFrame(docs)
                
                # Add human-readable columns
                if not df.empty:
                    df['filename'] = df['object_name'].apply(lambda x: x.split('/')[-1])
                    df['size'] = df['text_length'].apply(lambda x: f"{x:,} chars" if x else "N/A")
                    df['processed'] = pd.to_datetime(df['processed_at']).dt.strftime('%Y-%m-%d %H:%M')
                    
                    # Display table
                    display_df = df[['filename', 'content_type', 'size', 'processed']].copy()
                    display_df.columns = ['Filename', 'Type', 'Text Size', 'Processed At']
                    
                    st.dataframe(display_df, use_container_width=True)
                    
                    # Document selection for actions
                    st.subheader("Document Actions")
                    selected_doc = st.selectbox(
                        "Select a document:",
                        options=df['object_name'].tolist(),
                        format_func=lambda x: x.split("/")[-1]
                    )
                    
                    if selected_doc:
                        col1, col2, col3 = st.columns(3)
                        
                        with col1:
                            if st.button("View Details"):
                                show_document_results(selected_doc)
                        
                        with col2:
                            if st.button("Chat About This Doc"):
                                st.session_state.selected_doc_for_chat = selected_doc
                                st.info("Document selected for chat context")
                        
                        with col3:
                            if st.button("Reprocess"):
                                st.info("Reprocessing not implemented yet")
            else:
                st.info("No documents found. Upload some documents first!")
                
        else:
            st.error(f"Error loading document library: {response.text}")
            
    except Exception as e:
        st.error(f"Error loading document library: {str(e)}")

def show_system_health():
    st.header("🏥 System Health")
    
    if st.button("Refresh Health Status", type="primary"):
        with st.spinner("Checking system health..."):
            try:
                response = requests.get(f"{API_BASE_URL}/health")
                
                if response.status_code == 200:
                    health_data = response.json()
                    
                    # Overall status
                    status = health_data['status']
                    if status == 'healthy':
                        st.success(f"🟢 System Status: {status.upper()}")
                    else:
                        st.error(f"🔴 System Status: {status.upper()}")
                    
                    st.write(f"**Last Updated:** {health_data['timestamp']}")
                    
                    # Service status
                    st.subheader("Service Status")
                    
                    services = health_data['services']
                    
                    cols = st.columns(len(services))
                    for i, (service, status) in enumerate(services.items()):
                        with cols[i]:
                            if status == "healthy":
                                st.success(f"✅ {service.title()}")
                            else:
                                st.error(f"❌ {service.title()}")
                                st.caption(status)
                    
                    # Detailed service information
                    st.subheader("Service Details")
                    service_df = pd.DataFrame(list(services.items()), columns=['Service', 'Status'])
                    st.dataframe(service_df, use_container_width=True)
                    
                    # System metrics (if available)
                    st.subheader("System Information")
                    col1, col2, col3 = st.columns(3)
                    
                    with col1:
                        st.metric("API Server", "Running", "Port 8000")
                    with col2:
                        st.metric("Frontend", "Running", "Port 8501")
                    with col3:
                        st.metric("Nginx Proxy", "Running", "Port 80")
                    
                else:
                    st.error(f"Failed to get health status: {response.text}")
                    
            except Exception as e:
                st.error(f"Error checking system health: {str(e)}")

if __name__ == "__main__":
    main()
EOF

Step 12: Create Systemd Services for Applications
# Exit ai-system user to become root
exit

# Create systemd service for FastAPI backend
sudo tee /etc/systemd/system/ai-backend.service > /dev/null << 'EOF'
[Unit]
Description=AI Document Processing Backend (FastAPI)
After=network.target postgresql-16.service kafka.service redis.service minio.service ollama.service
Wants=postgresql-16.service kafka.service redis.service minio.service ollama.service

[Service]
Type=simple
User=ai-system
Group=ai-system
WorkingDirectory=/opt/ai-system
Environment=PATH=/opt/ai-system/venv/bin
Environment=PYTHONPATH=/opt/ai-system
ExecStart=/opt/ai-system/venv/bin/python apps/backend/main.py
Restart=always
RestartSec=10
StandardOutput=append:/opt/ai-system/logs/backend.log
StandardError=append:/opt/ai-system/logs/backend-error.log

[Install]
WantedBy=multi-user.target
EOF

# Create systemd service for Streamlit frontend
sudo tee /etc/systemd/system/ai-frontend.service > /dev/null << 'EOF'
[Unit]
Description=AI Document Processing Frontend (Streamlit)
After=network.target ai-backend.service
Wants=ai-backend.service

[Service]
Type=simple
User=ai-system
Group=ai-system
WorkingDirectory=/opt/ai-system
Environment=PATH=/opt/ai-system/venv/bin
Environment=PYTHONPATH=/opt/ai-system
ExecStart=/opt/ai-system/venv/bin/streamlit run apps/frontend/main.py --server.port 8501 --server.address 0.0.0.0
Restart=always
RestartSec=10
StandardOutput=append:/opt/ai-system/logs/frontend.log
StandardError=append:/opt/ai-system/logs/frontend-error.log

[Install]
WantedBy=multi-user.target
EOF

# Create log files and set permissions
sudo mkdir -p /opt/ai-system/logs
sudo chown ai-system:ai-system /opt/ai-system/logs
sudo touch /opt/ai-system/logs/{backend,backend-error,frontend,frontend-error}.log
sudo chown ai-system:ai-system /opt/ai-system/logs/*.log

# Enable and start services
sudo systemctl daemon-reload
sudo systemctl enable ai-backend ai-frontend
sudo systemctl start ai-backend
sleep 10
sudo systemctl start ai-frontend

# Check service status
sudo systemctl status ai-backend ai-frontend

Step 13: Create Configuration and Management Scripts
# Create environment configuration file
sudo tee /opt/ai-system/config/environment.env > /dev/null << 'EOF'
# Database Configuration
DATABASE_URL=postgresql://ai_user:AiUserPassword123@localhost/ai_document_db
DATABASE_POOL_SIZE=20
DATABASE_MAX_OVERFLOW=0

# Redis Configuration
REDIS_URL=redis://localhost:6379
REDIS_DB=0
REDIS_TIMEOUT=5

# Kafka Configuration
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
KAFKA_CONSUMER_GROUP=ai-processors
KAFKA_AUTO_OFFSET_RESET=earliest

# MinIO Configuration
MINIO_ENDPOINT=localhost:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=MinioPassword123
MINIO_SECURE=false

# Ollama Configuration
OLLAMA_HOST=http://localhost:11434
OLLAMA_TIMEOUT=120
DEFAULT_MODEL=llama3.1:8b
EMBEDDING_MODEL=nomic-embed-text

# Application Configuration
MAX_FILE_SIZE=100MB
ALLOWED_EXTENSIONS=pdf,docx,doc,txt,html,xml,csv,xlsx,pptx,jpg,jpeg,png
TEMP_DIR=/opt/ai-system/temp
EOF

# Create startup script
sudo tee /opt/ai-system/scripts/start-all.sh > /dev/null << 'EOF'
#!/bin/bash

echo "Starting AI Document Processing System..."

# Start infrastructure services
echo "Starting infrastructure services..."
sudo systemctl start postgresql-16
sudo systemctl start zookeeper
sleep 5
sudo systemctl start kafka
sudo systemctl start redis
sudo systemctl start minio
sudo systemctl start ollama

# Wait for services to be ready
echo "Waiting for services to be ready..."
sleep 15

# Start application services
echo "Starting application services..."
sudo systemctl start ai-backend
sleep 10
sudo systemctl start ai-frontend

# Start web server
echo "Starting web server..."
sudo systemctl start nginx

echo "All services started successfully!"
echo "Access the application at: http://localhost"
echo "API documentation at: http://localhost/api/docs"
echo "MinIO console at: http://localhost/minio-console"
EOF

# Create status check script
sudo tee /opt/ai-system/scripts/check-status.sh > /dev/null << 'EOF'
#!/bin/bash

echo "AI Document Processing System Status Check"
echo "=========================================="

services=("postgresql-16" "zookeeper" "kafka" "redis" "minio" "ollama" "ai-backend" "ai-frontend" "nginx")

for service in "${services[@]}"; do
    if systemctl is-active --quiet $service; then
        echo "✅ $service: RUNNING"
    else
        echo "❌ $service: STOPPED"
    fi
done

echo ""
echo "Port Status:"
echo "============"
netstat -tlnp 2>/dev/null | grep -E ':(5432|2181|9092|6379|9000|9001|11434|8000|8501|80) ' | while read line; do
    port=$(echo $line | awk '{print $4}' | sed 's/.*://')
    case $port in
        5432) echo "✅ PostgreSQL: $port" ;;
        2181) echo "✅ Zookeeper: $port" ;;
        9092) echo "✅ Kafka: $port" ;;
        6379) echo "✅ Redis: $port" ;;
        9000) echo "✅ MinIO API: $port" ;;
        9001) echo "✅ MinIO Console: $port" ;;
        11434) echo "✅ Ollama: $port" ;;
        8000) echo "✅ FastAPI Backend: $port" ;;
        8501) echo "✅ Streamlit Frontend: $port" ;;
        80) echo "✅ Nginx: $port" ;;
    esac
done

echo ""
echo "Quick Health Check:"
echo "==================="
curl -s http://localhost:8000/health | python3 -m json.tool 2>/dev/null || echo "❌ Backend API not responding"
EOF

# Create log monitoring script
sudo tee /opt/ai-system/scripts/monitor-logs.sh > /dev/null << 'EOF'
#!/bin/bash

echo "AI Document Processing System Logs"
echo "==================================="

if [ "$1" = "backend" ]; then
    tail -f /opt/ai-system/logs/backend.log
elif [ "$1" = "frontend" ]; then
    tail -f /opt/ai-system/logs/frontend.log
elif [ "$1" = "errors" ]; then
    tail -f /opt/ai-system/logs/backend-error.log /opt/ai-system/logs/frontend-error.log
else
    echo "Usage: $0 [backend|frontend|errors]"
    echo "Available log files:"
    ls -la /opt/ai-system/logs/
fi
EOF

# Make scripts executable
sudo chmod +x /opt/ai-system/scripts/*.sh
sudo chown -R ai-system:ai-system /opt/ai-system/scripts

Step 14: System Testing and Verification
# Test all services are running
sudo /opt/ai-system/scripts/check-status.sh

# Test API endpoints
echo "Testing API health endpoint..."
curl -s http://localhost:8000/health | python3 -m json.tool

# Test Streamlit frontend
echo "Testing Streamlit frontend..."
curl -s -I http://localhost:8501

# Test through Nginx proxy
echo "Testing Nginx proxy..."
curl -s -I http://localhost/

# Test MinIO
echo "Testing MinIO..."
curl -s http://localhost:9000/minio/health/live

# Test Ollama
echo "Testing Ollama..."
curl -s http://localhost:11434/api/tags

# Test document upload (with a sample file)
echo "Creating test document..."
echo "This is a test document for the AI processing system." > /tmp/test.txt

echo "Testing document upload..."
curl -X POST -F "file=@/tmp/test.txt" http://localhost:8000/upload-document

# Clean up test file
rm /tmp/test.txt

Step 15: Configure Firewall and Security
# Configure firewall rules
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=8000/tcp  # Remove in production
sudo firewall-cmd --permanent --add-port=8501/tcp  # Remove in production
sudo firewall-cmd --reload

# Set up log rotation
sudo tee /etc/logrotate.d/ai-document-system > /dev/null << 'EOF'
/opt/ai-system/logs/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 0644 ai-system ai-system
    postrotate
        systemctl reload ai-backend ai-frontend
    endscript
}
EOF

# Create backup script
sudo tee /opt/ai-system/scripts/backup.sh > /dev/null << 'EOF'
#!/bin/bash

BACKUP_DIR="/opt/ai-system/backups/$(date +%Y-%m-%d)"
mkdir -p $BACKUP_DIR

# Backup database
sudo -u postgres pg_dump ai_document_db > $BACKUP_DIR/database.sql

# Backup MinIO data
sudo -u minio cp -r /var/lib/minio $BACKUP_DIR/

# Backup configuration
cp -r /opt/ai-system/config $BACKUP_DIR/

# Backup logs
cp -r /opt/ai-system/logs $BACKUP_DIR/

echo "Backup completed: $BACKUP_DIR"
EOF

sudo chmod +x /opt/ai-system/scripts/backup.sh

# Set up daily backup cron job
echo "0 2 * * * /opt/ai-system/scripts/backup.sh" | sudo crontab -u ai-system -

Usage Guide and Testing
Step 16: Complete System Test
# Start all services
sudo /opt/ai-system/scripts/start-all.sh

# Wait for services to fully initialize
sleep 30

# Check system status
sudo /opt/ai-system/scripts/check-status.sh

# Access the application
echo ""
echo "🎉 Installation Complete!"
echo "========================="
echo ""
echo "Access URLs:"
echo "- Main Application: http://localhost"
echo "- API Documentation: http://localhost/api/docs"
echo "- MinIO Console: http://localhost/minio-console"
echo "- Direct Streamlit: http://localhost:8501"
echo "- Direct FastAPI: http://localhost:8000"
echo ""
echo "Default Credentials:"
echo "- MinIO: minioadmin / MinioPassword123"
echo ""
echo "Management Commands:"
echo "- Check Status: sudo /opt/ai-system/scripts/check-status.sh"
echo "- View Logs: sudo /opt/ai-system/scripts/monitor-logs.sh [backend|frontend|errors]"
echo "- Backup System: sudo /opt/ai-system/scripts/backup.sh"
echo ""
echo "Service Management:"
echo "- Start All: sudo /opt/ai-system/scripts/start-all.sh"
echo "- Restart Backend: sudo systemctl restart ai-backend"
echo "- Restart Frontend: sudo systemctl restart ai-frontend"
echo ""

Troubleshooting Guide
Common Issues and Solutions
    1 Services not starting
# Check service logs
sudo journalctl -u ai-backend -f
sudo journalctl -u ai-frontend -f

# Check dependencies
sudo systemctl status postgresql-16 kafka redis minio ollama

    2 Port conflicts
# Check what's using ports
sudo netstat -tlnp | grep -E ':(8000|8501|9092|5432|6379|9000|11434)'

# Kill conflicting processes if needed
sudo fuser -k 8000/tcp

    3 Permission issues
# Fix ownership
sudo chown -R ai-system:ai-system /opt/ai-system

# Fix permissions
sudo chmod -R 755 /opt/ai-system
sudo chmod -R 644 /opt/ai-system/logs

    4 Database connection issues
# Test database connection
sudo -u postgres psql -c "SELECT version();"

# Reset database password
sudo -u postgres psql -c "ALTER USER ai_user PASSWORD 'AiUserPassword123';"

    5 Document processing errors
# Check document processing logs
sudo /opt/ai-system/scripts/monitor-logs.sh backend

# Test Ollama models
sudo -u ollama ollama list
sudo -u ollama ollama run llama3.1:8b "Test message"

Performance Optimization
System Tuning
# Increase file limits for high document throughput
echo 'ai-system soft nofile 65536' | sudo tee -a /etc/security/limits.conf
echo 'ai-system hard nofile 65536' | sudo tee -a /etc/security/limits.conf

# Optimize PostgreSQL for document processing
sudo tee -a /var/lib/pgsql/16/data/postgresql.conf << 'EOF'
# AI Document Processing optimizations
shared_buffers = 512MB
effective_cache_size = 2GB
work_mem = 8MB
maintenance_work_mem = 128MB
checkpoint_completion_target = 0.9
wal_level = minimal
max_connections = 100
EOF

sudo systemctl restart postgresql-16

# Optimize Redis for caching
sudo tee -a /etc/redis.conf << 'EOF'
# Document processing cache optimization
maxmemory 512mb
maxmemory-policy allkeys-lru
EOF

sudo systemctl restart redis

Conclusion
This comprehensive installation guide provides a complete AI document processing system on RHEL with the following features:
✅ Local LLM inference with Ollama (Llama 3.1, CodeLlama, Mistral)
✅ Multi-format document processing with Unstructured.io and Docling
✅ Scalable messaging with Apache Kafka
✅ Robust database with PostgreSQL 16
✅ High-performance caching with Redis
✅ S3-compatible object storage with MinIO
✅ Modern web interfaces with FastAPI and Streamlit
✅ Production-ready deployment with Nginx and systemd
✅ Comprehensive monitoring and logging
✅ Security hardening and backup procedures
The system is designed to handle enterprise workloads while maintaining security and reliability standards required for production environments. All components are configured to work together seamlessly and can be scaled horizontally as needed.
For production deployment, consider implementing additional security measures such as SSL/TLS certificates, network segmentation, and advanced monitoring solutions based on your specific requirements.
⁂
