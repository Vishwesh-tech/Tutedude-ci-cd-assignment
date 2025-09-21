Sir as you can see my ip addess is change in both backend and frontend because while creating frontend pipeline 
my ubuntu ec2 connection was broken due to which i has to stop start the instance again which give me a different 
public ipv4 address that's its change in backend and frontnend



Flask Pileline Script :

pipeline {
    agent any
    
    environment {
        APP_NAME = 'flask-app'
        APP_PORT = '5000'
        PYTHON_ENV = '/var/lib/jenkins/flask-venv'
    }
    
    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }
        
        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Vishwesh-tech/Tutedude-Docker-Assigment.git'
            }
        }
        
        stage('Verify Repository Structure') {
            steps {
                script {
                    sh '''
                        echo "=== Repository Contents ==="
                        ls -la
                        
                        echo "=== Flask App Directory Check ==="
                        if [ -d "flask-app" ]; then
                            echo "✅ flask-app directory exists"
                            ls -la flask-app/
                        else
                            echo "⚠️ flask-app directory not found - creating it"
                            mkdir -p flask-app
                        fi
                    '''
                }
            }
        }
        
        stage('Create Flask App if Missing') {
            steps {
                script {
                    sh '''
                        cd flask-app
                        
                        # Create app.py if it doesn't exist
                        if [ ! -f "app.py" ]; then
                            echo "Creating Flask application..."
                            cat > app.py << 'EOF'
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return jsonify({
                        'message': 'Hello from Flask CI/CD!',
                        'status': 'success',
                        'service': 'flask-app',
                        'version': '1.0',
                        'deployed_via': 'Jenkins Pipeline'
                    })

@app.route('/health')
def health_check():
    return jsonify({
                        'status': 'healthy',
                        'service': 'flask-app',
                        'uptime': 'available'
                    }), 200

@app.route('/info')
def info():
    return jsonify({
                        'app': 'Flask Demo',
                        'deployed_via': 'Jenkins CI/CD',
                        'pm2_managed': True,
                        'python_version': '3.x'
                    })

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port, debug=False)
EOF
                        fi
                        
                        # Create requirements.txt if it doesn't exist
                        if [ ! -f "requirements.txt" ]; then
                            echo "Flask==2.3.3" > requirements.txt
                        fi
                        
                        echo "=== Flask App Files ==="
                        ls -la
                    '''
                }
            }
        }
        
        stage('Setup Python Environment') {
            steps {
                script {
                    sh '''
                        cd flask-app
                        
                        echo "=== Python Environment Setup ==="
                        # Remove existing virtual environment
                        rm -rf venv
                        
                        # Create virtual environment
                        python3 -m venv venv
                        
                        # Activate and install dependencies
                        . venv/bin/activate
                        pip install --upgrade pip
                        pip install -r requirements.txt
                        
                        # Verify Flask installation
                        python -c "import flask; print('Flask version:', flask.__version__)"
                    '''
                }
            }
        }
        
        stage('Test Application') {
            steps {
                script {
                    sh '''
                        cd flask-app
                        . venv/bin/activate
                        
                        echo "=== Testing Flask Application ==="
                        # Start Flask app in background for testing
                        python app.py &
                        FLASK_PID=$!
                        echo "Flask test PID: $FLASK_PID"
                        
                        # Wait for Flask to start
                        sleep 8
                        
                        # Test the application endpoints
                        echo "Testing root endpoint..."
                        curl -s http://localhost:5000/ || echo "Root endpoint test failed"
                        
                        echo "Testing health endpoint..."
                        curl -s http://localhost:5000/health || echo "Health endpoint test failed"
                        
                        # Kill the test Flask process
                        kill $FLASK_PID || true
                        sleep 2
                        
                        echo "Flask application test completed"
                    '''
                }
            }
        }
        
        stage('Stop Existing Application') {
            steps {
                script {
                    sh '''
                        echo "=== Stopping Existing Flask Application ==="
                        
                        # Stop PM2 process if exists
                        pm2 stop ${APP_NAME} || echo "No existing ${APP_NAME} process"
                        pm2 delete ${APP_NAME} || echo "No process to delete"
                        
                        # Kill any Python processes on port 5000
                        pkill -f "python.*flask" || echo "No flask processes found"
                        lsof -ti:${APP_PORT} | xargs kill -9 || echo "Port ${APP_PORT} is free"
                        
                        sleep 3
                        pm2 list
                    '''
                }
            }
        }
        
        stage('Deploy Application') {
            steps {
                script {
                    sh '''
                        cd flask-app
                        
                        echo "=== Deploying Flask Application ==="
                        
                        # Get absolute path to virtual environment Python
                        VENV_PYTHON=$(pwd)/venv/bin/python
                        echo "Using Python interpreter: $VENV_PYTHON"
                        
                        # Start application with PM2
                        pm2 start app.py \\
                            --name ${APP_NAME} \\
                            --interpreter $VENV_PYTHON \\
                            --cwd $(pwd) \\
                            --time
                        
                        # Save PM2 configuration
                        pm2 save
                        
                        echo "=== PM2 Status ==="
                        pm2 list
                        pm2 show ${APP_NAME}
                    '''
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    sh '''
                        echo "=== Flask Application Health Check ==="
                        
                        # Wait for application to start
                        sleep 10
                        
                        # Check PM2 status
                        if pm2 show ${APP_NAME} | grep -q "online"; then
                            echo "✅ PM2 process is online"
                        else
                            echo "❌ PM2 process is not online"
                            pm2 logs ${APP_NAME} --lines 20
                            exit 1
                        fi
                        
                        # Quick health check with timeout
                        echo "Testing Flask application..."
                        if timeout 30 curl -f -s http://localhost:${APP_PORT}/; then
                            echo "✅ Flask app is responding!"
                            echo "Testing health endpoint..."
                            timeout 30 curl -f -s http://localhost:${APP_PORT}/health || echo "Health endpoint not available"
                            echo "🎉 Flask application deployed successfully!"
                        else
                            echo "❌ Flask app not responding"
                            pm2 logs ${APP_NAME} --lines 20
                            exit 1
                        fi
                    '''
                }
            }
        }
    }
    
    post {
        always {
            script {
                sh '''
                    echo "=== Flask Pipeline Summary ==="
                    echo "PM2 processes:"
                    pm2 list || true
                    
                    echo "Flask app logs (last 15 lines):"
                    pm2 logs ${APP_NAME} --lines 15 || true
                    
                    echo "System resources:"
                    free -h
                '''
            }
        }
        success {
            echo '🎉 Flask application pipeline completed successfully!'
            echo 'Access your Flask app at: http://your-server:5000'
        }
        failure {
            echo '❌ Flask application pipeline failed. Check logs above for details.'
        }
    }
}



Express Pileline Script : 


pipeline {
    agent any
    
    environment {
        APP_NAME = 'express-app'
        APP_PORT = '3000'
        PATH = "/usr/local/bin:${PATH}"
    }
    
    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }
        
        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Vishwesh-tech/Tutedude-Docker-Assigment.git'
            }
        }
        
        stage('Verify Repository Structure') {
            steps {
                script {
                    sh '''
                        echo "=== Repository Contents ==="
                        ls -la
                        
                        echo "=== Frontend Directory Check ==="
                        if [ -d "frontend" ]; then
                            echo "✅ frontend directory exists"
                            ls -la frontend/
                        else
                            echo "⚠️ frontend directory not found - creating it"
                            mkdir -p frontend
                        fi
                        
                        echo "=== Node.js Files Check ==="
                        cd frontend
                        ls -la package.json || echo "package.json not found"
                        ls -la app.js || ls -la server.js || ls -la index.js || echo "No main JS file found"
                    '''
                }
            }
        }
        
        stage('Setup Application') {
            steps {
                script {
                    sh '''
                        cd frontend
                        
                        # Create package.json if it doesn't exist
                        if [ ! -f "package.json" ]; then
                            echo "Creating package.json..."
                            cat > package.json << 'EOF'
{
  "name": "frontend-express-app",
  "version": "1.0.0",
  "description": "Frontend Express App - CI/CD Demo",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "keywords": ["express", "nodejs", "frontend", "cicd"],
  "author": "Jenkins CI/CD Pipeline",
  "license": "MIT"
}
EOF
                        fi
                        
                        # Create app.js if no main file exists
                        if [ ! -f "app.js" ] && [ ! -f "server.js" ] && [ ! -f "index.js" ]; then
                            echo "Creating Express application..."
                            cat > app.js << 'EOF'
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(express.json());
app.use(express.static('public'));

// Routes
app.get('/', (req, res) => {
    res.json({
        message: 'Hello from Frontend Express CI/CD!',
        status: 'success',
        service: 'frontend-express',
        version: '1.0',
        deployed_via: 'Jenkins Pipeline',
        timestamp: new Date().toISOString()
    });
});

app.get('/health', (req, res) => {
    res.status(200).json({
        status: 'healthy',
        service: 'frontend-express',
        uptime: Math.floor(process.uptime()),
        memory: process.memoryUsage(),
        timestamp: new Date().toISOString()
    });
});

app.get('/info', (req, res) => {
    res.json({
        app: 'Frontend Express Demo',
        deployed_via: 'Jenkins CI/CD',
        pm2_managed: true,
        node_version: process.version,
        environment: process.env.NODE_ENV || 'production'
    });
});

// 404 handler
app.use('*', (req, res) => {
    res.status(404).json({
        status: 'error',
        message: 'Route not found',
        path: req.originalUrl
    });
});

// Error handling middleware
app.use((err, req, res, next) => {
    console.error(err.stack);
    res.status(500).json({
        status: 'error',
        message: 'Internal server error'
    });
});

// Start server
const server = app.listen(PORT, '0.0.0.0', () => {
    console.log(`Frontend Express app listening on port ${PORT}`);
    console.log(`Health check: http://localhost:${PORT}/health`);
    console.log(`Info: http://localhost:${PORT}/info`);
});

// Graceful shutdown
process.on('SIGTERM', () => {
    console.log('SIGTERM received, shutting down gracefully');
    server.close(() => {
        process.exit(0);
    });
});

process.on('SIGINT', () => {
    console.log('SIGINT received, shutting down gracefully');
    server.close(() => {
        process.exit(0);
    });
});
EOF
                        fi
                        
                        echo "=== Frontend App Files ==="
                        ls -la
                    '''
                }
            }
        }
        
        stage('Verify Node.js Environment') {
            steps {
                script {
                    sh '''
                        echo "=== Node.js Environment Check ==="
                        echo "Node.js version:"
                        node --version || echo "❌ Node.js not found"
                        
                        echo "NPM version:"
                        npm --version || echo "❌ NPM not found"
                        
                        echo "PM2 version:"
                        pm2 --version || echo "❌ PM2 not found"
                    '''
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                script {
                    sh '''
                        cd frontend
                        
                        echo "=== Installing Dependencies ==="
                        # Clean previous installations
                        rm -rf node_modules package-lock.json
                        
                        # Install dependencies
                        npm install
                        
                        # Verify installation
                        echo "=== Installed Packages ==="
                        npm list --depth=0
                        
                        # Test Express import
                        node -e "console.log('Express version:', require('express/package.json').version)"
                    '''
                }
            }
        }
        
        stage('Test Application') {
            steps {
                script {
                    sh '''
                        cd frontend
                        
                        echo "=== Testing Express Application ==="
                        # Find the main file
                        MAIN_FILE="app.js"
                        if [ -f "server.js" ]; then
                            MAIN_FILE="server.js"
                        elif [ -f "index.js" ] && [ ! -f "app.js" ]; then
                            MAIN_FILE="index.js"
                        fi
                        
                        echo "Using main file: $MAIN_FILE"
                        
                        # Test the application
                        node $MAIN_FILE &
                        APP_PID=$!
                        echo "Express test PID: $APP_PID"
                        
                        # Wait for app to start
                        sleep 8
                        
                        # Test endpoints
                        echo "Testing root endpoint:"
                        curl -s http://localhost:3000/ || echo "Root endpoint test failed"
                        
                        echo "Testing health endpoint:"
                        curl -s http://localhost:3000/health || echo "Health endpoint test failed"
                        
                        # Kill test process
                        kill $APP_PID || true
                        sleep 2
                        
                        echo "Express application test completed"
                    '''
                }
            }
        }
        
        stage('Stop Existing Application') {
            steps {
                script {
                    sh '''
                        echo "=== Stopping Existing Express Application ==="
                        
                        # Stop PM2 process if exists
                        pm2 stop ${APP_NAME} || echo "No existing ${APP_NAME} process"
                        pm2 delete ${APP_NAME} || echo "No process to delete"
                        
                        # Kill any Node processes on port 3000
                        pkill -f "node.*frontend" || echo "No frontend node processes found"
                        lsof -ti:${APP_PORT} | xargs kill -9 || echo "Port ${APP_PORT} is free"
                        
                        sleep 3
                        pm2 list
                    '''
                }
            }
        }
        
        stage('Deploy Application') {
            steps {
                script {
                    sh '''
                        cd frontend
                        
                        echo "=== Deploying Express Application ==="
                        
                        # Find the main file
                        MAIN_FILE="app.js"
                        if [ -f "server.js" ]; then
                            MAIN_FILE="server.js"
                        elif [ -f "index.js" ] && [ ! -f "app.js" ]; then
                            MAIN_FILE="index.js"
                        fi
                        
                        echo "Deploying with main file: $MAIN_FILE"
                        
                        # Start application with PM2
                        pm2 start $MAIN_FILE \\
                            --name ${APP_NAME} \\
                            --cwd $(pwd) \\
                            --time \\
                            --env NODE_ENV=production
                        
                        # Save PM2 configuration
                        pm2 save
                        
                        echo "=== PM2 Status ==="
                        pm2 list
                        pm2 show ${APP_NAME}
                    '''
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    sh '''
                        echo "=== Express Application Health Check ==="
                        
                        # Wait for application to start
                        sleep 10
                        
                        # Check PM2 status
                        if pm2 show ${APP_NAME} | grep -q "online"; then
                            echo "✅ PM2 process is online"
                        else
                            echo "❌ PM2 process is not online"
                            pm2 logs ${APP_NAME} --lines 20
                            exit 1
                        fi
                        
                        # Quick health check with timeout
                        echo "Testing Express application..."
                        if timeout 30 curl -f -s http://localhost:${APP_PORT}/; then
                            echo "✅ Express app is responding!"
                            echo "Testing health endpoint..."
                            timeout 30 curl -f -s http://localhost:${APP_PORT}/health || echo "Health endpoint not available"
                            echo "🎉 Express application deployed successfully!"
                        else
                            echo "❌ Express app not responding"
                            pm2 logs ${APP_NAME} --lines 20
                            exit 1
                        fi
                    '''
                }
            }
        }
    }
    
    post {
        always {
            script {
                sh '''
                    echo "=== Express Pipeline Summary ==="
                    echo "PM2 processes:"
                    pm2 list || true
                    
                    echo "Express app logs (last 15 lines):"
                    pm2 logs ${APP_NAME} --lines 15 || true
                    
                    echo "Port status:"
                    netstat -tlnp | grep :${APP_PORT} || echo "No process on port ${APP_PORT}"
                '''
            }
        }
        success {
            echo '🎉 Express application pipeline completed successfully!'
            echo 'Access your Express app at: http://your-server:3000'
        }
        failure {
            echo '❌ Express application pipeline failed. Check logs above for details.'
        }
    }
}