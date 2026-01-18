# AWS Deployment Guide

Hướng dẫn chi tiết để deploy ứng dụng Spring Boot lên AWS EC2 thông qua GitHub Actions CI/CD.

## Yêu Cầu Chuẩn Bị

### 1. AWS EC2 Instance
- Ubuntu 22.04 LTS (khuyến nghị)
- Tối thiểu: t2.medium (2 vCPU, 4GB RAM)
- Storage: 20GB trở lên
- Security Group cho phép:
  - Port 22 (SSH)
  - Port 8080 (Application)
  - Outbound: All traffic

### 2. MySQL Database
- AWS RDS MySQL hoặc MySQL trên EC2
- Database: `re_shopping_cart_db`
- Cho phép connection từ EC2 instance

### 3. Docker Hub Account
- Tạo repository: `re-shopping-cart-web-app`
- Generate access token

## Bước 1: Setup AWS EC2

### Cài Đặt Docker
```bash
# Update packages
sudo apt-get update
sudo apt-get upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER

# Start Docker
sudo systemctl enable docker
sudo systemctl start docker

# Verify installation
docker --version
```

### Cài Đặt Curl (cho health check)
```bash
sudo apt-get install -y curl
```

### Tạo Log Directory
```bash
sudo mkdir -p /var/log/re-shopping-cart
sudo chown $USER:$USER /var/log/re-shopping-cart
```

## Bước 2: Setup MySQL Database

### Import Database Schema
```bash
# Upload SQL file to EC2
scp -i your-key.pem src/main/resources/db/re_shopping_cart_db.sql ubuntu@EC2_IP:/tmp/

# Import to MySQL
mysql -h RDS_ENDPOINT -u USERNAME -p DATABASE_NAME < /tmp/re_shopping_cart_db.sql
```

### Verify Connection từ EC2
```bash
mysql -h RDS_ENDPOINT -u USERNAME -p -e "SHOW DATABASES;"
```

## Bước 3: Cấu Hình GitHub Secrets

Vào Repository → Settings → Secrets and variables → Actions → New repository secret

### Required Secrets

| Secret Name | Description | Example |
|------------|-------------|---------|
| `DOCKER_USER` | Docker Hub username | `your-dockerhub-username` |
| `DOCKER_TOKEN` | Docker Hub access token | `dckr_pat_xxxxx...` |
| `EC2_HOST` | EC2 public IP or hostname | `ec2-12-34-56-78.compute-1.amazonaws.com` |
| `EC2_USER` | EC2 SSH username | `ubuntu` hoặc `ec2-user` |
| `EC2_KEY` | EC2 SSH private key | Copy toàn bộ nội dung file `.pem` |
| `DB_URL` | MySQL JDBC URL | `jdbc:mysql://rds-endpoint:3306/re_shopping_cart_db?useSSL=true&serverTimezone=UTC` |
| `DB_USERNAME` | Database username | `admin` |
| `DB_PASSWORD` | Database password | `your-db-password` |

### Cách Lấy EC2_KEY
```bash
# Trên máy local, mở file SSH key
cat ~/.ssh/your-ec2-key.pem

# Copy toàn bộ nội dung bao gồm:
# -----BEGIN RSA PRIVATE KEY-----
# ...
# -----END RSA PRIVATE KEY-----
```

## Bước 4: Deploy Application

### Automatic Deployment
Push code lên branch `main`:
```bash
git add .
git commit -m "Deploy to AWS"
git push origin main
```

GitHub Actions sẽ tự động:
1. ✅ Build Maven project
2. ✅ Build Docker image
3. ✅ Push to Docker Hub
4. ✅ Deploy to EC2
5. ✅ Run health check

### Manual Deployment
Trên EC2, chạy thủ công:
```bash
# Pull latest image
docker pull your-dockerhub-username/re-shopping-cart-web-app:latest

# Stop old container
docker rm -f re-shopping-cart-web-app

# Run new container
docker run -d \
  --name re-shopping-cart-web-app \
  --restart unless-stopped \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e DB_URL='jdbc:mysql://your-rds:3306/re_shopping_cart_db?useSSL=true&serverTimezone=UTC' \
  -e DB_USERNAME='admin' \
  -e DB_PASSWORD='yourpassword' \
  -e SERVER_PORT=8080 \
  -e JAVA_OPTS="-Xms512m -Xmx1024m" \
  -e TZ=Asia/Ho_Chi_Minh \
  -v /var/log/re-shopping-cart:/app/logs \
  your-dockerhub-username/re-shopping-cart-web-app:latest
```

## Bước 5: Verification

### Check Application Status
```bash
# Container status
docker ps | grep re-shopping-cart

# View logs
docker logs -f re-shopping-cart-web-app

# Health check
curl http://localhost:8080/actuator/health
```

### Access Application
- **Homepage**: `http://EC2_PUBLIC_IP:8080`
- **Health Check**: `http://EC2_PUBLIC_IP:8080/actuator/health`
- **Metrics**: `http://EC2_PUBLIC_IP:8080/actuator/metrics`

### Expected Health Response
```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP"
    },
    "diskSpace": {
      "status": "UP"
    },
    "ping": {
      "status": "UP"
    }
  }
}
```

## Troubleshooting

### Issue: Container không start
```bash
# Check logs
docker logs re-shopping-cart-web-app

# Common issues:
# - Database connection failed → Check DB_URL, credentials
# - Port already in use → Kill process on port 8080
```

### Issue: Health check failed
```bash
# Exec into container
docker exec -it re-shopping-cart-web-app sh

# Test health endpoint inside container
curl http://localhost:8080/actuator/health

# Check database connection
curl http://localhost:8080/actuator/health/db
```

### Issue: Out of memory
```bash
# Increase JVM heap size in CI/CD workflow
-e JAVA_OPTS="-Xms512m -Xmx1024m"

# Or upgrade EC2 instance type
```

### View Application Logs
```bash
# On EC2
tail -f /var/log/re-shopping-cart/application.log

# Or Docker logs
docker logs --tail 100 -f re-shopping-cart-web-app
```

### Database Connection Issues
```bash
# Test MySQL connection from EC2
mysql -h RDS_ENDPOINT -u USERNAME -p

# Check security group rules
# - EC2 security group allows outbound to RDS
# - RDS security group allows inbound from EC2
```

## Security Best Practices

1. **SSH Keys**: Không share private key, rotate định kỳ
2. **Database**: Dùng RDS với SSL enabled
3. **Security Groups**: Chỉ mở ports cần thiết
4. **Secrets**: Không commit secrets vào git
5. **HTTPS**: Setup Load Balancer + SSL certificate cho production

## Monitoring & Maintenance

### Log Rotation
```bash
# Add to crontab
0 0 * * * find /var/log/re-shopping-cart -name "*.log" -mtime +30 -delete
```

### Auto-restart on Failure
Container đã có `--restart unless-stopped`, tự động restart khi crash.

### Backup Database
```bash
# Daily backup script
mysqldump -h RDS_ENDPOINT -u USERNAME -p DATABASE > backup_$(date +%Y%m%d).sql
```

## Next Steps

1. ✅ Setup Load Balancer cho high availability
2. ✅ Add Auto Scaling Group
3. ✅ Setup CloudWatch monitoring
4. ✅ Configure SSL/TLS certificates
5. ✅ Setup RDS read replicas
6. ✅ Implement Blue-Green deployment

---

**Support**: Nếu gặp vấn đề, check GitHub Actions logs hoặc EC2 system logs.
