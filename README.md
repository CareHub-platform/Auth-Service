# Auth-Service

# Microservices Project: Auth Service with OAuth Integration

## Project Overview

You're building an authentication microservice with OAuth integration, using Node.js/Express.js for the backend and Next.js for the frontend. Here's a comprehensive plan:

## 1. Auth Microservice Architecture

### Tech Stack:
- **Backend**: Node.js + Express.js
- **Frontend**: Next.js
- **Database**: PostgreSQL (recommended) or MongoDB
- **OAuth Providers**: Google, GitHub, Facebook (at minimum)
- **Authentication**: JWT (JSON Web Tokens)
- **Security**: bcrypt for password hashing, helmet, rate limiting

## 2. Database Schema (PostgreSQL Example)

```sql
-- Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    is_verified BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- OAuth providers table
CREATE TABLE oauth_providers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    client_id VARCHAR(255) NOT NULL,
    client_secret VARCHAR(255) NOT NULL,
    redirect_uri VARCHAR(255) NOT NULL,
    authorization_url VARCHAR(255) NOT NULL,
    token_url VARCHAR(255) NOT NULL,
    user_info_url VARCHAR(255) NOT NULL,
    scopes VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- User OAuth accounts
CREATE TABLE user_oauth_accounts (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    provider_id INTEGER REFERENCES oauth_providers(id) ON DELETE CASCADE,
    provider_user_id VARCHAR(255) NOT NULL,
    access_token VARCHAR(255),
    refresh_token VARCHAR(255),
    expires_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, provider_id),
    UNIQUE(provider_id, provider_user_id)
);

-- Verification tokens
CREATE TABLE verification_tokens (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    token VARCHAR(255) UNIQUE NOT NULL,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Password reset tokens
CREATE TABLE password_reset_tokens (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    token VARCHAR(255) UNIQUE NOT NULL,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Refresh tokens
CREATE TABLE refresh_tokens (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    token VARCHAR(255) UNIQUE NOT NULL,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

## 3. OAuth Integration Implementation

### Backend (Node.js/Express.js) Structure:

```
/auth-service
├── config/               # Configuration files
│   ├── oauth.config.js   # OAuth provider configurations
│   └── db.config.js      # Database configuration
├── controllers/          # Route controllers
│   ├── auth.controller.js
│   └── oauth.controller.js
├── middlewares/          # Custom middlewares
│   ├── auth.middleware.js
│   └── oauth.middleware.js
├── models/               # Database models
│   ├── user.model.js
│   ├── oauth.model.js
│   └── token.model.js
├── routes/               # API routes
│   ├── auth.routes.js
│   └── oauth.routes.js
├── services/             # Business logic
│   ├── auth.service.js
│   ├── oauth.service.js
│   └── token.service.js
├── utils/                # Utility functions
│   ├── jwt.js
│   ├── oauth.js
│   └── email.js
├── app.js                # Express app setup
└── server.js             # Server entry point
```

### Key OAuth Implementation Files:

1. **oauth.config.js**:
```javascript
module.exports = {
  google: {
    clientId: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    redirectUri: process.env.GOOGLE_REDIRECT_URI,
    authorizationUrl: 'https://accounts.google.com/o/oauth2/v2/auth',
    tokenUrl: 'https://oauth2.googleapis.com/token',
    userInfoUrl: 'https://www.googleapis.com/oauth2/v3/userinfo',
    scopes: ['profile', 'email']
  },
  github: {
    clientId: process.env.GITHUB_CLIENT_ID,
    clientSecret: process.env.GITHUB_CLIENT_SECRET,
    redirectUri: process.env.GITHUB_REDIRECT_URI,
    authorizationUrl: 'https://github.com/login/oauth/authorize',
    tokenUrl: 'https://github.com/login/oauth/access_token',
    userInfoUrl: 'https://api.github.com/user',
    scopes: ['user:email']
  }
  // Add other providers as needed
};
```

2. **oauth.service.js**:
```javascript
const axios = require('axios');
const { OAuthProvider, User } = require('../models');
const { generateJWT } = require('../utils/jwt');

class OAuthService {
  async getProviderConfig(providerName) {
    return OAuthProvider.findOne({ where: { name: providerName } });
  }

  async handleOAuthCallback(providerName, code) {
    const providerConfig = await this.getProviderConfig(providerName);
    
    // Exchange code for access token
    const { access_token } = await this.exchangeCodeForToken(
      providerConfig.tokenUrl,
      {
        client_id: providerConfig.clientId,
        client_secret: providerConfig.clientSecret,
        redirect_uri: providerConfig.redirectUri,
        code,
        grant_type: 'authorization_code'
      }
    );

    // Get user info from provider
    const userInfo = await this.getUserInfo(
      providerConfig.userInfoUrl,
      access_token
    );

    // Find or create user
    const user = await this.findOrCreateUser(providerName, userInfo);

    // Generate JWT tokens
    return generateJWT(user);
  }

  async exchangeCodeForToken(url, data) {
    const response = await axios.post(url, data, {
      headers: { Accept: 'application/json' }
    });
    return response.data;
  }

  async getUserInfo(url, accessToken) {
    const response = await axios.get(url, {
      headers: { Authorization: `Bearer ${accessToken}` }
    });
    return response.data;
  }

  async findOrCreateUser(providerName, userInfo) {
    // Implementation depends on your user model and provider data
    // Typically involves checking if user exists by email or provider ID
    // and creating if not exists
  }
}

module.exports = new OAuthService();
```

## 4. Frontend (Next.js) Implementation

### Key Pages and Components:

1. **Login Page with OAuth Options** (`pages/login.js`):
```jsx
import { signIn } from 'next-auth/react';

export default function LoginPage() {
  return (
    <div className="login-container">
      <h1>Login</h1>
      <form>{/* Email/password form */}</form>
      
      <div className="oauth-providers">
        <button onClick={() => signIn('google')}>
          Sign in with Google
        </button>
        <button onClick={() => signIn('github')}>
          Sign in with GitHub
        </button>
        {/* Add more providers as needed */}
      </div>
    </div>
  );
}
```

2. **OAuth Callback Page** (`pages/api/auth/[...nextauth].js`):
```javascript
import NextAuth from 'next-auth';
import GoogleProvider from 'next-auth/providers/google';
import GitHubProvider from 'next-auth/providers/github';

export default NextAuth({
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    }),
    GitHubProvider({
      clientId: process.env.GITHUB_CLIENT_ID,
      clientSecret: process.env.GITHUB_CLIENT_SECRET,
    }),
  ],
  callbacks: {
    async jwt({ token, user, account }) {
      if (account) {
        token.accessToken = account.access_token;
        token.provider = account.provider;
      }
      return token;
    },
    async session({ session, token }) {
      session.accessToken = token.accessToken;
      session.provider = token.provider;
      return session;
    },
  },
});
```

## 5. Environment Variables

Create a `.env` file with these variables:

```
# Database
DB_HOST=localhost
DB_PORT=5432
DB_USER=youruser
DB_PASSWORD=yourpassword
DB_NAME=auth_service

# JWT
JWT_SECRET=your_jwt_secret
JWT_EXPIRES_IN=1d
REFRESH_TOKEN_EXPIRES_IN=7d

# OAuth Providers
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
GOOGLE_REDIRECT_URI=http://localhost:3000/api/auth/callback/google

GITHUB_CLIENT_ID=your_github_client_id
GITHUB_CLIENT_SECRET=your_github_client_secret
GITHUB_REDIRECT_URI=http://localhost:3000/api/auth/callback/github

# Email (for verification)
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=your_email@example.com
SMTP_PASSWORD=your_email_password
EMAIL_FROM=no-reply@yourapp.com
```

## 6. Key Features to Implement

1. **Local Authentication**:
   - Email/password registration and login
   - Email verification
   - Password reset

2. **OAuth Authentication**:
   - Google, GitHub, Facebook login
   - Account linking (allow users to connect multiple OAuth providers)
   - Token refresh handling

3. **Security**:
   - CSRF protection
   - Rate limiting
   - Secure cookie settings
   - Input validation

4. **API Endpoints**:
   - POST /auth/register - User registration
   - POST /auth/login - User login
   - POST /auth/refresh - Refresh token
   - GET /auth/oauth/[provider] - Initiate OAuth flow
   - GET /auth/oauth/[provider]/callback - OAuth callback
   - POST /auth/verify-email - Verify email
   - POST /auth/forgot-password - Request password reset
   - POST /auth/reset-password - Reset password

## 7. Deployment Considerations

1. **Containerization**:
   - Dockerize both auth service and frontend
   - Use Docker Compose for local development

2. **Scaling**:
   - Consider Redis for session storage in production
   - Implement database connection pooling

3. **Monitoring**:
   - Add logging middleware
   - Implement health check endpoints

Would you like me to elaborate on any specific part of this architecture or provide more detailed code examples for any component?
