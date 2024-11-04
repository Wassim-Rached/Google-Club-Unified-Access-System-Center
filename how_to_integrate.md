Integrating with the **Google Club Unified Access System (UAS)** for authentication and authorization is a straightforward process. Here’s a step-by-step guide to get your app fully integrated with Google Club UAS.

For any assistance, please contact the admins at [contact.isetchggl@gmail.com](mailto:contact.isetchggl@gmail.com).

---

## Google UAS Authentication and Authorization Flow

This diagram explains how the flow works from the client until the access of the protected resource.

![Google UAS Authentication and Authorization Flow](./images/uas-cycle.png)

---

## 1. Getting Started

To authenticate with UAS, you'll need the **public key** for JWT verification, which one of the system admins will provide. This public key will allow your app to verify the JWT tokens issued by the **Central Authentication Service (CAS)**.

### 1.1 Acquiring the JWT Token

1. **Request the JWT Token**: From your frontend client, send a request to the CAS server’s login endpoint to obtain the JWT token.
2. **Attach the Token**: Include the token as a `Bearer {token}` in the Authorization header for all requests where authentication is required.

> **Note**: You only need to implement the login page for authentication in your frontend. Other functionalities like signup or password reset are already handled by AMWC.

---

## 2. Setting Up Middleware for Authentication

On the backend, implement a middleware to verify the JWT token in each incoming request.

- The middleware will check the Authorization header for the JWT.
- Verify the JWT's signature using the public key.
- If verification fails, return a `401 Unauthorized` response.

If the token is optional and verification fails, simply skip further authentication checks. If the token is required and verification succeeds, the user is authenticated.

### JWT Format

The JWT (JSON Web Token) used for authentication will have the following format:

#### Header

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

#### Payload

```json
{
  "email": "user@example.com",
  "sub": "123e4567-e89b-12d3-a456-426614174000"
}
```

#### Signature

The signature is created using the RS256 algorithm and the provided public key.

#### Complete JWT Example

```plaintext
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6InVzZXJAZXhhbXBsZS5jb20iLCJzdWIiOiJ1c2VyX2lkIn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### Middleware Example in Express.js (TypeScript)

Here's an example middleware in **Express.js** with TypeScript that handles JWT verification.

```typescript
import { Request, Response, NextFunction } from "express";
import jwt from "jsonwebtoken";

const publicKey = `YOUR_PUBLIC_KEY_HERE`; // Replace with the provided public key

export const authenticateJWT =
  (required: boolean = false) =>
  (req: Request, res: Response, next: NextFunction) => {
    const authHeader = req.headers.authorization;

    if (!authHeader) {
      if (required) {
        return res.status(401).json({ message: "Token required" });
      }
      return next(); // Skip if token is not required
    }

    const token = authHeader.split(" ")[1];

    try {
      const decoded = jwt.verify(token, publicKey);
      req.user = decoded; // Store decoded token in request object
      next();
    } catch (error) {
      if (required) {
        return res.status(401).json({ message: "Invalid token" });
      }
      next();
    }
  };
```

Use this middleware in your routes to protect endpoints by passing `required: true` if the token is mandatory.

---

## 3. Adding Authorization

If your application requires authorization, you can query the CAS server for a list of authorities assigned to the user.

1. **Request Authorities**: Use the endpoint in CAS with the JWT in the header to retrieve a list of authorities (permissions) as strings.
2. **Manage Access**: Based on the authorities list, manage access to various parts of your application as per your authorization logic.

### Example: Fetching Authorities from CAS

Here's a minimal example of fetching authorities from the CAS server and saving them in memory.

```typescript
import axios from "axios";

let authorities: string[] = [];
const casUrl = env.casUrl;

const fetchAuthorities = async (jwtToken: string) => {
  try {
    const response = await axios.get(`${casUrl}/api/authorities`, {
      headers: {
        Authorization: `Bearer ${jwtToken}`,
      },
    });
    authorities = response.data;
  } catch (error) {
    console.error("Error fetching authorities:", error);
  }
};
```

### Example: Middleware for Managing Access

This middleware checks if the user has the required authority to access a route.

```typescript
import { Request, Response, NextFunction } from "express";

const checkAuthority = (requiredAuthority: string) => {
  return (req: Request, res: Response, next: NextFunction) => {
    // get the authoriteis from wherever you have putten them
    const userAuthorities: string[] = req.user?.authorities || [];

    if (userAuthorities.includes(requiredAuthority)) {
      return next();
    }

    return res.status(403).json({ message: "Forbidden" });
  };
};

// Usage example
app.get("/protected-route", checkAuthority("docs.perm.example"), (req, res) => {
  res.send("You have access to this route");
});
```

### Caching Authorities

UAS offers two ways to handle authorities:

- **Fetch on Every Request**: Guarantees up-to-date access control but may affect performance.
- **Cache Locally**: Improves performance by reducing API calls but may delay authority updates. You can cache the authorities in a local variable or any caching layer of your choice.

Choose the approach that best balances security and performance for your application.

---

## 4. Informing ICS of Your Service’s Authorities

To help manage permissions, you should inform ICS of the authorities your service uses. Implement an endpoint at `/api/authorities` that provides a list of authorities used in your application.

1. **Purpose**: The Accounts Console Web Client (ACWC) will periodically call this endpoint and send the authorities list to ICS for management.
2. **Dynamic Generation**: Generate this endpoint dynamically from a centralized file or variable containing all authorities used in your service to ensure consistency.
3. **Admin Notification**: Inform the admins as they need to update certain variables to include your service as an integrated app. For any assistance, please contact the admins at [contact.isetchggl@gmail.com](mailto:contact.isetchggl@gmail.com).

### Example

Define a centralized list of authorities in your service and expose it via the `/api/authorities` endpoint. This way, any changes to authorities only need to be updated in one place.

```typescript
export const AUTHORITIES = {
  view_dashboard: "docs.perm.view_dashboard",
  edit_profile: "docs.perm.edit_profile",
  delete_user: "docs.perm.delete_user",
};

app.get("/api/authorities", (req: Request, res: Response) => {
  const authorities: string[] = Object.values(AUTHORITIES);
  res.json(authorities);
});
```

---

> **Note**: If you encounter any issues while integrating with UAS, feel free to reach out to the founder, [Wassim Rached](https://wassimrached.me), for support. If you have any questions, please contact the admins at [contact.isetchggl@gmail.com](mailto:contact.isetchggl@gmail.com).
