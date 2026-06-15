# ShopVerse

A modern e-commerce application built with React and Go.

## Features

- User authentication (login/register)
- Product browsing and search
- Shopping cart management
- Order placement and history
- Wishlist functionality
- Admin dashboard for products, orders, users, coupons, reviews, notifications, and settings

## Tech Stack

**Frontend**
- React 18 + Vite
- Tailwind CSS
- React Router DOM
- Axios

**Backend**
- Go 1.24
- Fiber framework
- MySQL + GORM
- JWT authentication

## Project Structure

```
.
├── frontend/           # React frontend application
├── backend/            # Go backend API server
├── docker-compose.yml  # Docker orchestration
└── .env.example      # Environment template
```

## Development

### Prerequisites
- Node.js 18+
- Go 1.24+
- Docker & Docker Compose
- MySQL 8.0+

### Environment Setup

1. Copy `.env.example` to `.env`
2. Update values as needed for your environment

### Run with Docker

```bash
docker-compose up --build
```

Services:
- Frontend: http://localhost:11001
- Backend API: http://localhost:11000 (internal: 8080)
- MySQL: localhost:3306

### Local Development

**Frontend**
```bash
cd frontend
npm install
npm run dev
```

**Backend**
```bash
cd backend
go mod tidy
go run cmd/main.go
```

## API Endpoints

- `GET /health` - Health check
- `POST /api/auth/register` - User registration
- `POST /api/auth/login` - User login
- `GET /api/products` - List products (supports `category` and `search` query params)
- `GET /api/products/:id` - Get product by ID
- `POST /api/products` - Create product (protected)
- `GET /api/cart` - Get cart (protected)
- `POST /api/cart` - Add to cart (protected)
- `PUT /api/cart/:id` - Update cart item (protected)
- `DELETE /api/cart/:id` - Remove cart item (protected)
- `GET /api/orders` - List orders (protected)
- `POST /api/orders` - Create order (protected)

## License

MIT

## CI/CD

GitHub Actions workflows handle:
- Security scanning with Trivy
- Docker image builds and pushes to Amazon ECR
- Automatic deployment to Kubernetes via ArgoCD