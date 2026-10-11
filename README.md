# AWS Three-Tier Architecture with Redis Caching | DevOps Project

## 1. Project Overview

Designed and deployed a three-tier web application on AWS using Amazon EC2, Nginx, React.js, Node.js, Amazon ElastiCache for Redis, and Amazon RDS MySQL.

This project demonstrates practical DevOps skills, including Linux server administration, application deployment, reverse proxy configuration, process management, cloud networking, database connectivity, and performance optimization through Redis caching.

### Project Objectives

- Deploy a full-stack application on Amazon EC2.
- Configure Nginx as a web server and reverse proxy.
- Deploy and manage a Node.js backend using PM2.
- Serve a React production build through Nginx.
- Integrate Amazon RDS MySQL for persistent storage.
- Implement Redis caching to reduce repeated database queries.
- Test application connectivity, cache hits, cache misses, and database updates.

## 2. Architecture

![AWS Three-Tier Architecture](screenshots/architecture.png)

### Architecture Diagram

```text
                         User
                           |
                           v
                        Internet
                           |
                           v
                     Amazon EC2
                           |
                           v
                         Nginx
                      Port 80/443*
                           |
                +----------+----------+
                |                     |
                v                     v
         React Frontend         /api/* Requests
                                      |
                                      v
                              Node.js / Express
                                 Port 3000
                                      |
                              Check Redis Cache
                                      |
                           +----------+----------+
                           |                     |
                           v                     v
                       Cache Hit             Cache Miss
                           |                     |
                           |                     v
                           |                Amazon RDS
                           |                 MySQL
                           |                     |
                           |                     v
                           |               Store in Redis
                           |                     |
                           +----------+----------+
                                      |
                                      v
                                JSON Response
                                      |
                                      v
                                 React UI
```

*HTTPS on port 443 requires TLS configuration. The application configuration described in this project uses HTTP on port 80 unless HTTPS has been configured separately.*

### Request Flow

1. A user opens the website.
2. Nginx serves the React frontend.
3. React requests product data from `/api/products`.
4. Nginx forwards the API request to the Node.js backend.
5. Node.js checks Redis for cached product data.
6. If the data exists, Redis returns the cached result (**Cache Hit**).
7. If the data is missing or expired, Node.js queries Amazon RDS MySQL (**Cache Miss**).
8. The backend stores the query result in Redis with a 60-second expiration.
9. The backend returns the product data to React.
10. React displays the products on the webpage.

## 3. Technology Stack

| Technology | Purpose |
|---|---|
| Amazon EC2 | Compute and application hosting |
| Amazon Linux 2023 | Server operating system |
| Nginx | Web server and reverse proxy |
| React.js | Frontend user interface |
| Node.js | Backend runtime |
| Express.js | REST API framework |
| Amazon ElastiCache for Redis | In-memory caching |
| Amazon RDS MySQL | Persistent relational database |
| PM2 | Node.js process management |
| Axios | Frontend HTTP requests |
| Git | Version control |
| GitHub | Source code hosting |

## 4. Prerequisites

Before starting, ensure that you have:

- An AWS account.
- An Amazon EC2 instance running Amazon Linux 2023.
- SSH access through a terminal or MobaXterm.
- An Amazon RDS MySQL database.
- An Amazon ElastiCache for Redis instance.
- AWS security groups configured for the required connections.
- A GitHub account for source code management.

### Network Requirements

| Service | Port | Access |
|---|---:|---|
| SSH | 22 | Trusted administrator IPs |
| HTTP | 80 | Public web traffic |
| HTTPS | 443 | Public web traffic when TLS is configured |
| Node.js | 3000 | Preferably local/private access |
| Redis | 6379 | Authorized application resources |
| MySQL | 3306 | Authorized application resources |

Do not expose Redis or MySQL directly to the public internet. Configure security groups so that the application server can reach the database and cache privately.

## 5. Step 1 — Launch and Configure EC2

Launch an Amazon Linux 2023 EC2 instance with sufficient storage. This project used approximately 10 GB of storage.

Connect to the instance using SSH or MobaXterm.

Update the system packages:

```bash
sudo dnf update -y
```

Install Node.js and Git:

```bash
sudo dnf install nodejs git -y
```

Verify the installation:

```bash
node -v
npm -v
git --version
```

Use a supported Node.js LTS version compatible with your dependencies. If you specifically install Node.js 18, note that Node.js 18 has reached end of life; prefer a currently supported LTS release for a new deployment.

## 6. Step 2 — Create the Node.js Backend

Create the backend directory:

```bash
mkdir -p ~/backend
cd ~/backend
```

Initialize the Node.js project:

```bash
npm init -y
```

Install the dependencies:

```bash
npm install express mysql2 redis cors
```

### Backend Configuration

Create the backend file:

```bash
vi server.js
```

Use the following implementation:

```javascript
const express = require("express");
const mysql = require("mysql2/promise");
const redis = require("redis");
const cors = require("cors");

const app = express();

app.use(cors());

const redisClient = redis.createClient({
  url: process.env.REDIS_URL
});

redisClient.on("error", (err) => {
  console.error("Redis Error:", err);
});

const db = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME
});

async function startServer() {
  await redisClient.connect();

  app.get("/products", async (req, res) => {
    try {
      const cached = await redisClient.get("products");

      if (cached) {
        console.log("CACHE HIT");
        return res.json(JSON.parse(cached));
      }

      console.log("CACHE MISS");

      const [rows] = await db.query(
        "SELECT * FROM products"
      );

      await redisClient.setEx(
        "products",
        60,
        JSON.stringify(rows)
      );

      res.json(rows);
    } catch (err) {
      console.error(err);
      res.status(500).send("Error retrieving products");
    }
  });

  app.listen(3000, "127.0.0.1", () => {
    console.log("Backend running on port 3000");
  });
}

startServer().catch((err) => {
  console.error("Failed to start backend:", err);
  process.exit(1);
});
```

### Configure Environment Variables

Configure the connection settings in the EC2 environment rather than hardcoding credentials.

Example:

```bash
export DB_HOST="YOUR_RDS_ENDPOINT"
export DB_USER="YOUR_DB_USER"
export DB_PASSWORD="YOUR_DB_PASSWORD"
export DB_NAME="productsdb"
export REDIS_URL="redis://YOUR_REDIS_ENDPOINT:6379"
```

Replace the placeholders with your actual configuration.

These commands set variables for the current shell. For persistent PM2 deployment, configure the variables securely in the application's process environment or a protected environment file that is excluded from Git.

**Important:** The backend example binds to `127.0.0.1`, allowing Nginx to reach it locally without exposing port 3000 to the network. If your existing configuration binds to `0.0.0.0`, restrict inbound port 3000 access through the appropriate network controls.

## 7. Step 3 — Create the RDS MySQL Database

Install the MySQL-compatible client:

```bash
sudo dnf install mariadb105 -y
```

Connect to Amazon RDS:

```bash
mysql -h YOUR_RDS_ENDPOINT -u YOUR_DB_USER -p
```

Enter your database password when prompted.

Create the database:

```sql
CREATE DATABASE productsdb;

USE productsdb;
```

Create the products table:

```sql
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100),
    price INT
);
```

Insert sample data:

```sql
INSERT INTO products (name, price)
VALUES
('Laptop', 1000),
('Phone', 500),
('Keyboard', 100);
```

Add more sample products if needed:

```sql
INSERT INTO products (name, price)
VALUES
('Mouse', 150),
('Wire', 70),
('Microphone', 100);
```

Verify the records:

```sql
SELECT * FROM products;
```

Exit MySQL:

```sql
EXIT;
```

### Database Connectivity

The EC2 instance must be able to connect to the RDS endpoint on port 3306. Configure the RDS security group to permit MySQL traffic from the EC2 instance's security group.

## 8. Step 4 — Configure Redis ElastiCache

Amazon ElastiCache for Redis provides an in-memory caching layer between the backend and the database.

### Redis Configuration

The backend connects to Redis using the `REDIS_URL` environment variable.

For a non-TLS Redis endpoint:

```bash
export REDIS_URL="redis://YOUR_REDIS_ENDPOINT:6379"
```

If encryption in transit is enabled, configure the Redis client to use the appropriate TLS connection.

### Install the Redis CLI

Where the package is available for your Amazon Linux repository:

```bash
sudo dnf install redis6 -y
```

Verify the CLI:

```bash
redis6-cli --version
```

Connect to the cache:

```bash
redis6-cli -h YOUR_REDIS_ENDPOINT -p 6379
```

Use the appropriate TLS and authentication options if required by your ElastiCache configuration.

### Redis Caching Logic

The application uses:

- **Cache key:** `products`
- **Expiration:** 60 seconds
- **Cache Hit:** Return the cached product list.
- **Cache Miss:** Query MySQL and store the result in Redis.

Redis reduces repeated reads from RDS when the same product data is requested frequently.

## 9. Step 5 — Test the Backend

Navigate to the backend directory:

```bash
cd ~/backend
```

Start the backend temporarily:

```bash
node server.js
```

Open another terminal session in MobaXterm and run:

```bash
curl http://localhost:3000/products
```

Expected response:

```json
[
  {
    "id": 1,
    "name": "Laptop",
    "price": 1000
  },
  {
    "id": 2,
    "name": "Phone",
    "price": 500
  }
]
```

The exact response depends on the records in your database.

### Verify Cache Hits and Misses

Run the request multiple times:

```bash
curl http://localhost:3000/products
curl http://localhost:3000/products
```

The first request should normally show `CACHE MISS`, and the second request should show `CACHE HIT` if the cache has not expired or been cleared.

Stop the temporary server with:

```text
Ctrl + C
```

## 10. Step 6 — Configure PM2

PM2 manages the Node.js backend process and helps keep the application running after an SSH session ends.

Install PM2:

```bash
sudo npm install -g pm2
```

Start the backend:

```bash
cd ~/backend

pm2 start server.js --name backend
```

Check the process:

```bash
pm2 list
```

Save the process list:

```bash
pm2 save
```

Configure startup integration:

```bash
pm2 startup
```

Run the additional command printed by PM2, if prompted, then save the process list again:

```bash
pm2 save
```

View the backend logs:

```bash
pm2 logs backend
```

### Useful PM2 Commands

```bash
pm2 status
pm2 restart backend
pm2 stop backend
pm2 delete backend
pm2 logs backend
```

For production use, ensure the database and Redis environment variables are configured securely for the PM2 process. Test automatic recovery after a reboot.

## 11. Step 7 — Create the React Frontend

Navigate to your home directory:

```bash
cd ~
```

If the React application does not already exist, create it:

```bash
npx create-react-app frontend
cd frontend
npm install axios
```

### Frontend Code

Edit `src/App.js`:

```bash
vi src/App.js
```

Add the following code:

```javascript
import axios from "axios";
import { useEffect, useState } from "react";
import "./App.css";

function App() {
  const [products, setProducts] = useState([]);
  const [error, setError] = useState("");

  useEffect(() => {
    axios
      .get("/api/products")
      .then((res) => setProducts(res.data))
      .catch((err) => {
        console.error(err);
        setError("Unable to load products.");
      });
  }, []);

  return (
    <div className="container">
      <h1>AWS Product Store</h1>

      {error && <p>{error}</p>}

      <div className="grid">
        {products.map((product) => (
          <div className="card" key={product.id}>
            <h2>{product.name}</h2>
            <p>${product.price}</p>
          </div>
        ))}
      </div>
    </div>
  );
}

export default App;
```

Using `/api/products` allows the browser to send requests to the same origin that serves the frontend. Nginx forwards those requests to Node.js.

### Frontend CSS

Edit `src/App.css`:

```bash
vi src/App.css
```

Add the styling:

```css
body {
  margin: 0;
  background: #f4f6f9;
  font-family: Arial, sans-serif;
}

.container {
  padding: 40px;
}

h1 {
  text-align: center;
  color: #232f3e;
}

.grid {
  display: grid;
  grid-template-columns: repeat(
    auto-fit,
    minmax(250px, 1fr)
  );
  gap: 20px;
  margin-top: 40px;
}

.card {
  background: white;
  padding: 25px;
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}
```

## 12. Step 8 — Build the React Application

Navigate to the frontend directory:

```bash
cd ~/frontend
```

Create the production build:

```bash
npm run build
```

The production files will be generated in the `build/` directory.

These static files will be served by Nginx.

## 13. Step 9 — Install and Configure Nginx

Install Nginx:

```bash
sudo dnf install nginx -y
```

Enable and start the service:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

Check its status:

```bash
sudo systemctl status nginx
```

### Deploy the React Build

Copy the production files to the Nginx document root:

```bash
sudo cp -r ~/frontend/build/* /usr/share/nginx/html/
```

### Configure Nginx as a Reverse Proxy

Create the application configuration:

```bash
sudo vi /etc/nginx/conf.d/app.conf
```

Add:

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:3000/;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### How the Reverse Proxy Works

Without a reverse proxy, clients may need to access the Node.js backend directly on port 3000.

With Nginx:

- Nginx serves the React application.
- Nginx forwards API requests to Node.js.
- The backend can remain accessible only through the local interface.
- Nginx provides a central entry point for future HTTPS and routing configuration.

### Validate and Restart Nginx

```bash
sudo nginx -t
sudo systemctl restart nginx
```

If a default server configuration conflicts with `app.conf`, inspect the existing configuration before removing or disabling it.

### Deploy Frontend Updates

Whenever you modify the React source code:

```bash
cd ~/frontend
npm run build
sudo cp -r build/* /usr/share/nginx/html/
sudo systemctl reload nginx
```

## 14. Step 10 — Test the Complete Application

Open the EC2 public IP address in your browser:

```text
http://YOUR_EC2_PUBLIC_IP
```

Verify the following:

- Nginx serves the React frontend.
- The product list loads successfully.
- The `/api/products` endpoint returns JSON.
- The backend can connect to Redis and RDS.
- Repeated requests produce cache hits.
- Cache entries expire after 60 seconds.

Test the public API path through Nginx:

```bash
curl http://YOUR_EC2_PUBLIC_IP/api/products
```

Check the backend logs:

```bash
pm2 logs backend
```

## 15. Step 11 — Test Database Changes and Cache Expiration

Connect to MySQL from EC2:

```bash
mysql -h YOUR_RDS_ENDPOINT -u YOUR_DB_USER -p
```

Select the database:

```sql
USE productsdb;
```

Insert a new product:

```sql
INSERT INTO products (name, price)
VALUES ('Monitor', 250);
```

Refresh the website and monitor the backend logs:

```bash
pm2 logs backend
```

### Expected Behavior

1. If Redis contains the product list, the backend may return the cached version.
2. The newly inserted record may not appear until the cached entry expires.
3. After expiration, the backend queries MySQL again.
4. The new result is stored in Redis.
5. The next request can return the updated cached result.

The 60-second TTL limits how long the old product list remains cached. For a production system, implement cache invalidation when product records are created, updated, or deleted.

## 16. Troubleshooting

### Nginx Is Not Running

```bash
sudo nginx -t
sudo systemctl status nginx
sudo journalctl -u nginx --no-pager -n 50
```

### Backend Is Not Running

```bash
pm2 list
pm2 logs backend
```

### API Returns an Error

```bash
curl http://localhost:3000/products
curl http://localhost/api/products
```

Check the backend logs, environment variables, and Nginx configuration.

### Cannot Connect to RDS

Check:

- The RDS endpoint and database name.
- The database username and password.
- RDS security group inbound rules.
- Connectivity from EC2 to port 3306.

### Cannot Connect to Redis

Check:

- The Redis endpoint and port.
- ElastiCache security group rules.
- Whether EC2 and ElastiCache have appropriate network connectivity.
- Whether TLS or authentication is required.

### React Shows Old Data

The product list may be cached in Redis for 60 seconds. Wait for expiration or invalidate the cache after database changes.

## 17. Security Best Practices

- Never upload real database passwords or AWS credentials to GitHub.
- Store secrets in environment variables or a suitable secrets manager.
- Exclude `.env` files from Git.
- Restrict SSH access to trusted IP addresses.
- Keep Redis and RDS private and restrict access through security groups.
- Avoid exposing Node.js port 3000 publicly when Nginx can proxy requests locally.
- Configure HTTPS before using the application for production traffic.
- Use least-privilege IAM permissions.
- Keep the operating system and dependencies updated.

## 18. Future Enhancements

- CI/CD pipeline using GitHub Actions.
- Infrastructure as Code using Terraform.
- Docker containerization.
- Application Load Balancer and Auto Scaling.
- CloudWatch monitoring and alerting.
- HTTPS using a valid TLS certificate.
- Automated testing and deployment rollback.
- Cache invalidation and database connection monitoring.
- Centralized logging and security scanning.

## 19. Skills Demonstrated

**AWS | EC2 | Amazon Linux | Linux Administration | Nginx | Reverse Proxy | Node.js | React | Redis | Amazon RDS | MySQL | PM2 | Networking | Git | GitHub | Cloud Deployment**



This project was built to gain hands-on experience in AWS infrastructure, application deployment, Linux administration, reverse proxy configuration, caching, and DevOps fundamentals.
