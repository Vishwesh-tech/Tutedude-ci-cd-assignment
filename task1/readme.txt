FLASK & EXPRESS DEPLOYMENT ARCHITECTURE - EC2 SINGLE INSTANCE
================================================================

OVERVIEW:
---------
Both Flask (backend API) and Express (frontend) deployed on one EC2 instance
Cost-effective solution for small to medium applications

AWS INFRASTRUCTURE:
-------------------
- EC2 Instance: t2.micro (1 CPU, 1GB RAM, 8GB storage)
- Security Group Ports: 22 (SSH), 80 (HTTP), 3000 (Express), 5000 (Flask)
- Operating System: Amazon Linux 2

APPLICATION STACK:
------------------
Internet → Security Group → Nginx (Port 80) → Applications
                                   ↓
                          ┌────────┴────────┐
                          │   PM2 Manager   │
                          └────────┬────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
            ┌───────▼──────┐              ┌──────▼──────┐
            │  Express     │              │   Flask     │
            │  Frontend    │              │  Backend    │
            │  Port: 3000  │              │ Port: 5000  │
            │  Node.js     │              │  Python     │
            └──────────────┘              └─────────────┘

COMPONENT DETAILS:
------------------

1. NGINX REVERSE PROXY (Port 80):
   - Routes "/" to Express (3000)
   - Routes "/api/*" to Flask (5000)
   - Handles SSL termination
   - Serves static files

2. PM2 PROCESS MANAGER:
   - Monitors both applications
   - Auto-restart on crashes
   - Load balancing
   - Log management

3. FLASK BACKEND (Port 5000):
   - Python API server
   - Gunicorn WSGI server
   - RESTful endpoints
   - Database connections

4. EXPRESS FRONTEND (Port 3000):
   - Node.js web server
   - Static file serving
   - Frontend routing
   - Proxy to Flask API

REQUEST FLOW:
-------------
1. Client sends HTTP request to EC2 public IP
2. Security Group filters traffic (port 80 allowed)
3. Nginx receives request and routes based on URL:
   - Frontend requests (/) → Express server
   - API requests (/api/*) → Flask server
4. PM2 manages process health and monitoring
5. Application processes request and returns response
6. Response sent back through Nginx to client

TECHNOLOGY STACK:
-----------------
- OS: Amazon Linux 2
- Python: 3.x + Flask + Gunicorn
- Node.js: 18.x + Express
- Process Manager: PM2
- Reverse Proxy: Nginx
- Version Control: Git

PORTS CONFIGURATION:
--------------------
- 22: SSH access (your IP only)
- 80: HTTP traffic (public)
- 3000: Express direct access (optional)
- 5000: Flask direct access (optional)

BENEFITS:
---------
✓ Cost effective (single instance)
✓ Simple deployment and management
✓ Production ready with PM2 and Nginx
✓ Easy debugging (all services in one place)
✓ Fast setup time

LIMITATIONS:
------------
✗ Single point of failure
✗ Resource contention between services
✗ Limited horizontal scaling
✗ Shared security context

MONITORING:
-----------
- PM2 dashboard: pm2 status, pm2 logs
- Health endpoints: /health (Express), /api/health (Flask)
- System monitoring: htop, free -m, df -h
- Log files: ~/.pm2/logs/

DEPLOYMENT COMMANDS:
--------------------
# Start applications
pm2 start ecosystem.config.js

# Check status
pm2 status

# View logs
pm2 logs

# Restart apps
pm2 restart all

# System services
sudo systemctl status nginx
sudo systemctl restart nginx
