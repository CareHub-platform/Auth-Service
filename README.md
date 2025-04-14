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
---------------------------------------------------------

# Role-Based Authentication System with OAuth and Local Login

I'll expand the authentication system to include role-based access control (RBAC) while maintaining both local (email/password) and OAuth login options.

## Updated Database Schema (PostgreSQL)

```sql
-- Roles table
CREATE TABLE roles (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Insert default roles
INSERT INTO roles (name, description) VALUES 
('patient', 'Regular patient user'),
('doctor', 'Medical doctor'),
('lab_analyst', 'Laboratory analysis professional'),
('radiologist', 'Radiology center professional'),
('pharmacist', 'Medications company representative'),
('admin', 'System administrator');

-- Add role_id to users table
ALTER TABLE users ADD COLUMN role_id INTEGER REFERENCES roles(id) DEFAULT 1;

-- User permissions table (if you need granular permissions)
CREATE TABLE permissions (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Role-Permission mapping
CREATE TABLE role_permissions (
    role_id INTEGER REFERENCES roles(id) ON DELETE CASCADE,
    permission_id INTEGER REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);
```

## Updated User Model (models/user.model.js)

```javascript
const { DataTypes } = require('sequelize');
const bcrypt = require('bcrypt');

module.exports = (sequelize) => {
  const User = sequelize.define('User', {
    email: {
      type: DataTypes.STRING,
      allowNull: false,
      unique: true,
      validate: {
        isEmail: true
      }
    },
    password_hash: {
      type: DataTypes.STRING
    },
    first_name: {
      type: DataTypes.STRING,
      allowNull: false
    },
    last_name: {
      type: DataTypes.STRING,
      allowNull: false
    },
    is_verified: {
      type: DataTypes.BOOLEAN,
      defaultValue: false
    },
    is_active: {
      type: DataTypes.BOOLEAN,
      defaultValue: true
    },
    role_id: {
      type: DataTypes.INTEGER,
      defaultValue: 1, // Default to patient role
      references: {
        model: 'roles',
        key: 'id'
      }
    }
  }, {
    hooks: {
      beforeCreate: async (user) => {
        if (user.password_hash) {
          const salt = await bcrypt.genSalt(10);
          user.password_hash = await bcrypt.hash(user.password_hash, salt);
        }
      }
    }
  });

  User.associate = (models) => {
    User.belongsTo(models.Role, { foreignKey: 'role_id' });
    User.hasMany(models.UserOAuthAccount, { foreignKey: 'user_id' });
  };

  User.prototype.verifyPassword = async function(password) {
    return await bcrypt.compare(password, this.password_hash);
  };

  return User;
};
```

## Registration Flow with Role Selection

### Backend Controller (controllers/auth.controller.js)

```javascript
const { User, Role } = require('../models');
const { generateJWT } = require('../utils/jwt');
const { sendVerificationEmail } = require('../utils/email');

exports.register = async (req, res) => {
  try {
    const { email, password, first_name, last_name, role } = req.body;

    // Validate role exists
    const roleRecord = await Role.findOne({ where: { name: role } });
    if (!roleRecord) {
      return res.status(400).json({ error: 'Invalid role specified' });
    }

    // Check if user already exists
    const existingUser = await User.findOne({ where: { email } });
    if (existingUser) {
      return res.status(400).json({ error: 'Email already in use' });
    }

    // Create new user
    const user = await User.create({
      email,
      password_hash: password, // Will be hashed by model hook
      first_name,
      last_name,
      role_id: roleRecord.id
    });

    // Generate verification token and send email
    const verificationToken = generateVerificationToken(user.id);
    await sendVerificationEmail(user.email, verificationToken);

    // Respond without sensitive data
    const userData = user.get({ plain: true });
    delete userData.password_hash;

    res.status(201).json({
      message: 'Registration successful. Please check your email for verification.',
      user: userData
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

## Frontend Registration Form (Next.js)

```jsx
import { useState } from 'react';
import { useRouter } from 'next/router';

export default function RegisterPage() {
  const [formData, setFormData] = useState({
    email: '',
    password: '',
    firstName: '',
    lastName: '',
    role: 'patient'
  });
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');
  const router = useRouter();

  const handleChange = (e) => {
    setFormData({
      ...formData,
      [e.target.name]: e.target.value
    });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    setError('');

    try {
      const response = await fetch('/api/auth/register', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          email: formData.email,
          password: formData.password,
          first_name: formData.firstName,
          last_name: formData.lastName,
          role: formData.role
        })
      });

      const data = await response.json();

      if (!response.ok) {
        throw new Error(data.error || 'Registration failed');
      }

      // Redirect to verification page
      router.push('/verify-email?email=' + encodeURIComponent(formData.email));
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="register-container">
      <h1>Create Account</h1>
      {error && <div className="error-message">{error}</div>}
      
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label>Email</label>
          <input
            type="email"
            name="email"
            value={formData.email}
            onChange={handleChange}
            required
          />
        </div>
        
        <div className="form-group">
          <label>Password</label>
          <input
            type="password"
            name="password"
            value={formData.password}
            onChange={handleChange}
            required
            minLength="8"
          />
        </div>
        
        <div className="form-group">
          <label>First Name</label>
          <input
            type="text"
            name="firstName"
            value={formData.firstName}
            onChange={handleChange}
            required
          />
        </div>
        
        <div className="form-group">
          <label>Last Name</label>
          <input
            type="text"
            name="lastName"
            value={formData.lastName}
            onChange={handleChange}
            required
          />
        </div>
        
        <div className="form-group">
          <label>I am a</label>
          <select
            name="role"
            value={formData.role}
            onChange={handleChange}
            required
          >
            <option value="patient">Patient</option>
            <option value="doctor">Doctor</option>
            <option value="lab_analyst">Laboratory Analyst</option>
            <option value="radiologist">Radiology Specialist</option>
            <option value="pharmacist">Pharmacist</option>
          </select>
        </div>
        
        <button type="submit" disabled={loading}>
          {loading ? 'Creating account...' : 'Create Account'}
        </button>
      </form>
      
      <div className="oauth-options">
        <p>Or sign up with:</p>
        <button onClick={() => signIn('google')}>Google</button>
        <button onClick={() => signIn('github')}>GitHub</button>
      </div>
      
      <p className="login-link">
        Already have an account? <a href="/login">Log in</a>
      </p>
    </div>
  );
}
```

## Role-Based Middleware (middlewares/role.middleware.js)

```javascript
const { Role } = require('../models');

exports.requireRole = (roleNames) => {
  return async (req, res, next) => {
    try {
      // Get user from request (attached by auth middleware)
      const user = req.user;
      if (!user) {
        return res.status(401).json({ error: 'Unauthorized' });
      }

      // Get user's role
      const role = await Role.findByPk(user.role_id);
      if (!role) {
        return res.status(403).json({ error: 'Forbidden - Invalid role' });
      }

      // Check if user has required role
      if (Array.isArray(roleNames)) {
        if (!roleNames.includes(role.name)) {
          return res.status(403).json({ error: 'Forbidden - Insufficient permissions' });
        }
      } else if (role.name !== roleNames) {
        return res.status(403).json({ error: 'Forbidden - Insufficient permissions' });
      }

      // Attach role to request for downstream use
      req.role = role.name;
      next();
    } catch (error) {
      res.status(500).json({ error: error.message });
    }
  };
};
```

## Using the Role Middleware in Routes

```javascript
const express = require('express');
const router = express.Router();
const authController = require('../controllers/auth.controller');
const { authenticate } = require('../middlewares/auth.middleware');
const { requireRole } = require('../middlewares/role.middleware');

// Public routes
router.post('/register', authController.register);
router.post('/login', authController.login);

// Patient-only route example
router.get(
  '/patient/dashboard',
  authenticate,
  requireRole('patient'),
  authController.patientDashboard
);

// Doctor-only route example
router.get(
  '/doctor/dashboard',
  authenticate,
  requireRole('doctor'),
  authController.doctorDashboard
);

// Admin-only route example
router.get(
  '/admin/users',
  authenticate,
  requireRole('admin'),
  authController.listUsers
);

module.exports = router;
```

## OAuth Role Handling

When implementing OAuth, you'll need to determine the user's role. You can:

1. Ask for role during first-time OAuth login
2. Use email domain to infer role (e.g., @hospital.com → doctor)
3. Default to patient role and allow role upgrade later

Here's how to modify the OAuth callback handler:

```javascript
exports.oauthCallback = async (req, res) => {
  try {
    const { provider, code } = req.body;
    
    // Handle OAuth flow (get user info from provider)
    const oauthUser = await oauthService.handleOAuthFlow(provider, code);
    
    // Check if user exists
    let user = await User.findOne({
      where: { email: oauthUser.email },
      include: [Role]
    });

    // If new user, create with default patient role
    if (!user) {
      const patientRole = await Role.findOne({ where: { name: 'patient' } });
      user = await User.create({
        email: oauthUser.email,
        first_name: oauthUser.given_name || oauthUser.name.split(' ')[0],
        last_name: oauthUser.family_name || oauthUser.name.split(' ')[1] || '',
        is_verified: true, // OAuth emails are typically verified
        role_id: patientRole.id
      });
    }

    // Generate tokens
    const tokens = await generateJWT(user);

    res.json({
      user: {
        id: user.id,
        email: user.email,
        first_name: user.first_name,
        last_name: user.last_name,
        role: user.Role.name
      },
      ...tokens
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

## Additional Considerations

1. **Role-Specific Registration Fields**:
   - Doctors might need license numbers
   - Pharmacists might need company information
   - Consider separate registration flows or additional profile steps

2. **Role Change Requests**:
   - Implement admin approval for role upgrades
   - Add verification for professional roles (doctor, pharmacist, etc.)

3. **Profile Completion**:
   - Require additional information after registration based on role
   - Different dashboard views for each role

Would you like me to elaborate on any specific aspect, such as the role change approval process, professional verification, or the different dashboard implementations for each role?
