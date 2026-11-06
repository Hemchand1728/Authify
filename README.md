# Authify

Authify is a REST API backend for user authentication, built with Spring Boot. It handles registration, login with JWT, email OTP verification, and password reset. It has no frontend: any web or mobile client can call it.

## Features

- User registration with BCrypt-hashed passwords
- Login that returns a JWT (also set as an HttpOnly cookie)
- Email OTP account verification (valid 24 hours)
- Password reset via emailed OTP (valid 15 minutes)
- Stateless security with a custom JWT filter
- Swagger UI for exploring and testing the API
- Docker support

## Tech stack

- Java 21, Spring Boot 3.4
- Spring Web, Spring Security, Spring Data JPA, Validation
- MySQL
- JWT (jjwt)
- Spring Mail (SMTP)
- springdoc-openapi (Swagger UI)
- Lombok, Maven

## Project structure

```
src/main/java/in/sm/authify
├── config        Security and CORS configuration
├── controller    REST endpoints
├── entity        JPA entity (UserEntity)
├── excp          Global exception handler
├── filter        JWT request filter
├── io            Request and response DTOs
├── repository    Spring Data repository
├── service       Business logic and email sending
└── util          JWT helper
```

## Prerequisites

- Java 21
- MySQL running locally
- An SMTP account (for example Brevo) for sending emails

## Configuration

Secrets are read from environment variables, so none are stored in the repository.

| Variable        | Description                     | Default       |
|-----------------|---------------------------------|---------------|
| `DBHOST`        | MySQL host                      | `localhost`   |
| `DBPORT`        | MySQL port                      | `3306`        |
| `DBNAME`        | Database name                   | `authify_db`  |
| `DBUSER`        | Database user                   | `root`        |
| `DBPASS`        | Database password               | required      |
| `JWT_SECRET`    | Secret used to sign JWTs        | required      |
| `MAIL_USERNAME` | SMTP username                   | required      |
| `MAIL_PASSWORD` | SMTP password or key            | required      |
| `MAIL_FROM`     | Sender email address            | required      |

## Run locally

1. Create the database:

   ```bash
   mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS authify_db;"
   ```

2. Set the environment variables:

   ```bash
   export DBPASS='your-db-password'
   export JWT_SECRET='a-long-random-secret'
   export MAIL_USERNAME='your-smtp-username'
   export MAIL_PASSWORD='your-smtp-password'
   export MAIL_FROM='sender@example.com'
   ```

3. Start the app:

   ```bash
   ./mvnw spring-boot:run
   ```

The server starts on **http://localhost:8083**. Open Swagger UI at http://localhost:8083/swagger-ui/index.html.

Tables are created automatically on first start (`spring.jpa.hibernate.ddl-auto=update`).

## Run with Docker

```bash
docker build -t authify .
docker run -p 8083:8083 \
  -e DBHOST=host.docker.internal \
  -e DBPASS=your-db-password \
  -e JWT_SECRET=a-long-random-secret \
  -e MAIL_USERNAME=your-smtp-username \
  -e MAIL_PASSWORD=your-smtp-password \
  -e MAIL_FROM=sender@example.com \
  authify
```

## API endpoints

All endpoints are under `/api`.

| Method | Endpoint                | Auth     | Description                              |
|--------|-------------------------|----------|------------------------------------------|
| POST   | `/api/register`         | No       | Create an account                        |
| POST   | `/api/login`            | No       | Log in, returns JWT and sets cookie      |
| GET    | `/api/profile`          | Required | Get the current user's profile           |
| GET    | `/api/is-authenticated` | Required | Check whether the caller is logged in    |
| POST   | `/api/send-otp`         | Required | Email a verification OTP                 |
| POST   | `/api/verify-otp`       | Required | Verify the account with the OTP          |
| GET    | `/api/send-reset-otp`   | No       | Email a password reset OTP (`?email=`)   |
| POST   | `/api/reset-password`   | No       | Reset the password using the OTP         |

## Example usage

```bash
# Register
curl -X POST http://localhost:8083/api/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Test User","email":"test@example.com","password":"Password123"}'

# Log in and save the cookie
curl -c cookies.txt -X POST http://localhost:8083/api/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"Password123"}'

# Request and submit a verification OTP
curl -b cookies.txt -X POST http://localhost:8083/api/send-otp
curl -b cookies.txt -X POST http://localhost:8083/api/verify-otp \
  -H "Content-Type: application/json" \
  -d '{"otp":"1234"}'

# View the profile
curl -b cookies.txt http://localhost:8083/api/profile
```

## Authentication

Send the JWT either as a header (`Authorization: Bearer <token>`) or via the `jwt` cookie set at login. Tokens expire after 10 hours.
