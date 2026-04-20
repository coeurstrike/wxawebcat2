# AWS Deployment Guide for WebL8.AI

## 🎯 Recommended: AWS Lightsail (Easiest & Cheapest)

**Total Cost: ~$10-20/month**

AWS Lightsail is the simplest option - it's like a VPS with predictable pricing, no surprise bills, and includes everything you need.

---

## Option 1: Lightsail Container Service (Easiest)

### Cost: ~$10-20/month
- Nano: $7/month (512 MB RAM, 0.25 vCPU) - Dev/Testing
- Micro: $10/month (1 GB RAM, 0.5 vCPU) - Small production
- Small: $20/month (2 GB RAM, 1 vCPU) - Medium production

### Setup Steps

```bash
# 1. Install AWS CLI and Lightsail plugin
aws configure
aws lightsail create-container-service \
    --service-name wxacat \
    --power micro \
    --scale 1

# 2. Build and push your container
docker build -t wxacat .
aws lightsail push-container-image \
    --service-name wxacat \
    --label wxacat \
    --image wxacat

# 3. Deploy (see Lightsail console for container config)
```

### Database Options for Lightsail:
- **SQLite** (Free) - Good for <50K queries/day, file-based
- **Lightsail PostgreSQL** ($15/month) - Managed, automatic backups
- **Run PostgreSQL in container** ($0 extra) - Manual management

---

## Option 2: Single Lightsail Instance with Docker (Recommended)

### Cost: ~$5-10/month

This is the most cost-effective approach for small-medium workloads.

### Instance Pricing:
| Plan | RAM | vCPU | Storage | Price |
|------|-----|------|---------|-------|
| $3.50 | 512 MB | 1 | 20 GB | Testing only |
| $5 | 1 GB | 1 | 40 GB | Light production |
| **$10** | **2 GB** | **1** | **60 GB** | **Recommended** |
| $20 | 4 GB | 2 | 80 GB | High traffic |

### Step-by-Step Setup

#### 1. Create Lightsail Instance

```bash
# Via AWS Console:
# 1. Go to lightsail.aws.amazon.com
# 2. Create Instance → Linux → Ubuntu 22.04
# 3. Select $10/month plan (2 GB RAM)
# 4. Name it "wxacat-server"
# 5. Create!
```

#### 2. Connect and Install Docker

```bash
# SSH into your instance
ssh -i ~/.ssh/your-key.pem ubuntu@your-instance-ip

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu
sudo systemctl enable docker

# Install Docker Compose
sudo apt install docker-compose-plugin

# Logout and login again for group changes
exit
ssh -i ~/.ssh/your-key.pem ubuntu@your-instance-ip
```

#### 3. Deploy WebL8.AI

```bash
# Create app directory
mkdir -p ~/wxacat && cd ~/wxacat

# Upload your files (from local machine)
scp -i ~/.ssh/your-key.pem -r ./wxacat/* ubuntu@your-instance-ip:~/wxacat/

# Or clone from GitHub
git clone https://github.com/yourusername/wxacat.git
cd wxacat

# Create environment file
cat > .env << EOF
DB_PASSWORD=your_secure_password_here
OPENAI_API_KEY=sk-your-openai-key-here
EOF

# Start everything
docker compose up -d

# Check status
docker compose ps
docker compose logs -f
```

#### 4. Configure Firewall

```bash
# In Lightsail Console → Networking tab:
# Add rule: HTTPS (443)
# Add rule: Custom TCP 8000 (or use nginx for 80/443)
```

#### 5. (Optional) Setup Domain & SSL

```bash
# Install Nginx and Certbot
sudo apt install nginx certbot python3-certbot-nginx

# Configure Nginx
sudo nano /etc/nginx/sites-available/wxacat
```

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
# Enable site and get SSL
sudo ln -s /etc/nginx/sites-available/wxacat /etc/nginx/sites-enabled/
sudo certbot --nginx -d your-domain.com
sudo systemctl restart nginx
```

---

## Option 3: AWS Free Tier (First 12 Months Free)

### Cost: $0 for first year, then ~$15-30/month

If you have a new AWS account:

| Service | Free Tier | After Free Tier |
|---------|-----------|-----------------|
| EC2 t2.micro | 750 hrs/month | ~$8/month |
| RDS PostgreSQL | 750 hrs/month | ~$15/month |
| S3 | 5 GB | ~$0.02/GB |

### Setup with EC2

```bash
# 1. Launch EC2 instance (t2.micro or t3.micro)
# 2. Use Ubuntu 22.04 AMI
# 3. Configure security group (ports 22, 80, 443, 8000)
# 4. Follow same Docker setup as Lightsail above
```

---

## Option 4: Serverless (Lambda + API Gateway)

### Cost: ~$0-5/month for low traffic

Not recommended for this project because:
- Playwright/browser crawling doesn't work well in Lambda
- Cold starts affect response times
- More complex to set up

Better for: Static API responses, simple functions

---

## 📊 Cost Comparison

| Option | Monthly Cost | Setup Difficulty | Best For |
|--------|--------------|------------------|----------|
| **Lightsail $10** | **$10** | **Easy** | **Most users** |
| Lightsail $5 | $5 | Easy | Light usage |
| Lightsail + DB | $25 | Easy | High reliability |
| EC2 Free Tier | $0* | Medium | New AWS accounts |
| EC2 + RDS | $25+ | Medium | Enterprise |

*Free for 12 months

---

## 🔧 Production Checklist

### Before Launch:
- [ ] Change default database password
- [ ] Set strong OpenAI API key
- [ ] Configure proper logging
- [ ] Set up daily backups
- [ ] Enable Lightsail snapshots ($0.05/GB/month)

### Security:
- [ ] Use HTTPS (SSL certificate)
- [ ] Restrict SSH access to your IP
- [ ] Use strong API keys for customers
- [ ] Enable CloudWatch monitoring (optional)

### Scaling (if needed later):
- [ ] Upgrade to larger Lightsail plan (easy resize)
- [ ] Add Lightsail load balancer ($18/month)
- [ ] Migrate to ECS/Fargate for auto-scaling

---

## 🚀 Quick Start Script

Save this as `deploy.sh` and run on your Lightsail instance:

```bash
#!/bin/bash
set -e

# Configuration
OPENAI_KEY="sk-your-key-here"
DB_PASS="your-secure-password"

# Install Docker
if ! command -v docker &> /dev/null; then
    curl -fsSL https://get.docker.com | sh
    sudo usermod -aG docker $USER
    echo "Docker installed. Please logout/login and run again."
    exit 0
fi

# Create project directory
mkdir -p ~/wxacat && cd ~/wxacat

# Create .env file
cat > .env << EOF
DB_PASSWORD=$DB_PASS
OPENAI_API_KEY=$OPENAI_KEY
EOF

# Clone or update code
if [ -d ".git" ]; then
    git pull
else
    git clone https://github.com/yourusername/wxacat.git .
fi

# Start services
docker compose down 2>/dev/null || true
docker compose up -d --build

# Show status
echo ""
echo "✅ WebL8.AI is running!"
echo "   API: http://$(curl -s ifconfig.me):8000"
echo "   Docs: http://$(curl -s ifconfig.me):8000/api/docs"
echo ""
docker compose ps
```

---

## 💡 Tips

1. **Start small** - $10 Lightsail handles most use cases
2. **Use SQLite first** - Simpler, no extra cost, migrate to PostgreSQL later if needed
3. **Enable snapshots** - $1-2/month for peace of mind
4. **Monitor costs** - Set up billing alerts in AWS Console

---

## Need More Scale?

For 100K+ queries/day, consider:
- Multiple Lightsail instances with load balancer
- AWS ECS with Fargate
- Dedicated RDS PostgreSQL instance

Contact us for enterprise deployment assistance.
