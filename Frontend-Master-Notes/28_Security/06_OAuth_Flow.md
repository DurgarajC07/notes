# OAuth 2.0 & OAuth 2.1 Flow - Complete Guide

## Core Concepts

OAuth 2.0 is an authorization framework that enables applications to obtain limited access to user accounts on an HTTP service. It works by delegating user authentication to the service that hosts the user account and authorizing third-party applications to access that account.

### Key Terminology

- **Resource Owner**: The user who owns the data
- **Client**: The application requesting access to resources
- **Authorization Server**: Issues access tokens after authenticating the resource owner
- **Resource Server**: Hosts the protected resources
- **Access Token**: Credentials used to access protected resources
- **Refresh Token**: Used to obtain new access tokens
- **Scope**: Permissions requested by the client
- **Grant Type**: Method used by the client to obtain an access token

## OAuth 2.0 Grant Types

### 1. Authorization Code Flow (Most Secure)

The standard OAuth 2.0 flow for web applications with a backend server.

```typescript
// OAuth Configuration
interface OAuthConfig {
  clientId: string;
  clientSecret: string;
  authorizationEndpoint: string;
  tokenEndpoint: string;
  redirectUri: string;
  scope: string[];
}

// Authorization Server Configuration
class AuthorizationServer {
  private config: OAuthConfig;
  private authorizationCodes: Map<string, AuthCodeData> = new Map();
  private accessTokens: Map<string, TokenData> = new Map();

  constructor(config: OAuthConfig) {
    this.config = config;
  }

  // Step 1: Generate authorization URL
  generateAuthorizationUrl(state: string): string {
    const params = new URLSearchParams({
      response_type: "code",
      client_id: this.config.clientId,
      redirect_uri: this.config.redirectUri,
      scope: this.config.scope.join(" "),
      state: state,
    });

    return `${this.config.authorizationEndpoint}?${params.toString()}`;
  }

  // Step 2: Handle authorization callback
  async handleAuthorizationCallback(
    code: string,
    state: string,
  ): Promise<void> {
    // Verify state to prevent CSRF
    if (!this.verifyState(state)) {
      throw new Error("Invalid state parameter");
    }

    // Store authorization code temporarily
    const authCodeData: AuthCodeData = {
      code,
      expiresAt: Date.now() + 600000, // 10 minutes
      used: false,
    };

    this.authorizationCodes.set(code, authCodeData);
  }

  // Step 3: Exchange authorization code for tokens
  async exchangeCodeForTokens(code: string): Promise<TokenResponse> {
    const authCodeData = this.authorizationCodes.get(code);

    if (!authCodeData) {
      throw new Error("Invalid authorization code");
    }

    if (authCodeData.used) {
      throw new Error("Authorization code already used");
    }

    if (Date.now() > authCodeData.expiresAt) {
      throw new Error("Authorization code expired");
    }

    // Mark code as used (one-time use)
    authCodeData.used = true;

    // Generate tokens
    const accessToken = this.generateAccessToken();
    const refreshToken = this.generateRefreshToken();

    const tokenData: TokenData = {
      accessToken,
      refreshToken,
      expiresIn: 3600, // 1 hour
      scope: this.config.scope,
      tokenType: "Bearer",
    };

    this.accessTokens.set(accessToken, tokenData);

    return {
      access_token: accessToken,
      refresh_token: refreshToken,
      expires_in: tokenData.expiresIn,
      token_type: tokenData.tokenType,
      scope: tokenData.scope.join(" "),
    };
  }

  private generateAccessToken(): string {
    return crypto.randomBytes(32).toString("hex");
  }

  private generateRefreshToken(): string {
    return crypto.randomBytes(32).toString("hex");
  }

  private verifyState(state: string): boolean {
    // Implement state verification logic
    return true;
  }
}

interface AuthCodeData {
  code: string;
  expiresAt: number;
  used: boolean;
}

interface TokenData {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
  scope: string[];
  tokenType: string;
}

interface TokenResponse {
  access_token: string;
  refresh_token: string;
  expires_in: number;
  token_type: string;
  scope: string;
}
```

### 2. Authorization Code Flow with PKCE

PKCE (Proof Key for Code Exchange) adds an additional security layer, essential for public clients (SPAs, mobile apps).

```typescript
// PKCE Implementation
class PKCEAuthFlow {
  private codeVerifier: string;
  private codeChallenge: string;
  private codeChallengeMethod: "S256" | "plain" = "S256";

  constructor() {
    this.codeVerifier = this.generateCodeVerifier();
    this.codeChallenge = this.generateCodeChallenge(this.codeVerifier);
  }

  // Generate cryptographically random string
  private generateCodeVerifier(): string {
    const array = new Uint8Array(32);
    crypto.getRandomValues(array);
    return this.base64URLEncode(array);
  }

  // Generate code challenge from verifier
  private async generateCodeChallenge(verifier: string): Promise<string> {
    if (this.codeChallengeMethod === "plain") {
      return verifier;
    }

    // S256: BASE64URL(SHA256(verifier))
    const encoder = new TextEncoder();
    const data = encoder.encode(verifier);
    const hash = await crypto.subtle.digest("SHA-256", data);
    return this.base64URLEncode(new Uint8Array(hash));
  }

  private base64URLEncode(buffer: Uint8Array): string {
    const str = String.fromCharCode(...buffer);
    return btoa(str).replace(/\+/g, "-").replace(/\//g, "_").replace(/=/g, "");
  }

  // Step 1: Build authorization URL with PKCE
  async buildAuthorizationUrl(
    config: OAuthConfig,
    state: string,
  ): Promise<string> {
    const codeChallenge = await this.generateCodeChallenge(this.codeVerifier);

    const params = new URLSearchParams({
      response_type: "code",
      client_id: config.clientId,
      redirect_uri: config.redirectUri,
      scope: config.scope.join(" "),
      state: state,
      code_challenge: codeChallenge,
      code_challenge_method: this.codeChallengeMethod,
    });

    return `${config.authorizationEndpoint}?${params.toString()}`;
  }

  // Step 2: Exchange code with PKCE verifier
  async exchangeCodeWithPKCE(
    config: OAuthConfig,
    code: string,
  ): Promise<TokenResponse> {
    const body = new URLSearchParams({
      grant_type: "authorization_code",
      code: code,
      redirect_uri: config.redirectUri,
      client_id: config.clientId,
      code_verifier: this.codeVerifier, // PKCE proof
    });

    const response = await fetch(config.tokenEndpoint, {
      method: "POST",
      headers: {
        "Content-Type": "application/x-www-form-urlencoded",
      },
      body: body.toString(),
    });

    if (!response.ok) {
      throw new Error("Token exchange failed");
    }

    return await response.json();
  }

  getCodeVerifier(): string {
    return this.codeVerifier;
  }

  async getCodeChallenge(): Promise<string> {
    return this.codeChallenge;
  }
}
```

### 3. Client Credentials Flow (Server-to-Server)

Used for machine-to-machine authentication without user interaction.

```typescript
// Client Credentials Grant
class ClientCredentialsFlow {
  private config: OAuthConfig;

  constructor(config: OAuthConfig) {
    this.config = config;
  }

  async getAccessToken(): Promise<TokenResponse> {
    const credentials = btoa(
      `${this.config.clientId}:${this.config.clientSecret}`,
    );

    const body = new URLSearchParams({
      grant_type: "client_credentials",
      scope: this.config.scope.join(" "),
    });

    const response = await fetch(this.config.tokenEndpoint, {
      method: "POST",
      headers: {
        "Content-Type": "application/x-www-form-urlencoded",
        Authorization: `Basic ${credentials}`,
      },
      body: body.toString(),
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(`Token request failed: ${error.error_description}`);
    }

    return await response.json();
  }
}
```

### 4. Refresh Token Flow

Obtain new access tokens without re-authentication.

```typescript
// Refresh Token Handler
class RefreshTokenManager {
  private config: OAuthConfig;
  private refreshToken: string | null = null;
  private accessToken: string | null = null;
  private expiresAt: number = 0;

  constructor(config: OAuthConfig) {
    this.config = config;
  }

  setTokens(
    accessToken: string,
    refreshToken: string,
    expiresIn: number,
  ): void {
    this.accessToken = accessToken;
    this.refreshToken = refreshToken;
    this.expiresAt = Date.now() + expiresIn * 1000;
  }

  async getValidAccessToken(): Promise<string> {
    if (this.isTokenValid()) {
      return this.accessToken!;
    }

    if (!this.refreshToken) {
      throw new Error("No refresh token available");
    }

    return await this.refreshAccessToken();
  }

  private isTokenValid(): boolean {
    if (!this.accessToken) return false;
    // Refresh 5 minutes before expiry
    return Date.now() < this.expiresAt - 300000;
  }

  private async refreshAccessToken(): Promise<string> {
    const body = new URLSearchParams({
      grant_type: "refresh_token",
      refresh_token: this.refreshToken!,
      client_id: this.config.clientId,
      client_secret: this.config.clientSecret,
    });

    const response = await fetch(this.config.tokenEndpoint, {
      method: "POST",
      headers: {
        "Content-Type": "application/x-www-form-urlencoded",
      },
      body: body.toString(),
    });

    if (!response.ok) {
      throw new Error("Token refresh failed");
    }

    const tokenData: TokenResponse = await response.json();

    this.setTokens(
      tokenData.access_token,
      tokenData.refresh_token || this.refreshToken!,
      tokenData.expires_in,
    );

    return this.accessToken!;
  }

  clearTokens(): void {
    this.accessToken = null;
    this.refreshToken = null;
    this.expiresAt = 0;
  }
}
```

## Complete OAuth 2.1 Client Implementation

OAuth 2.1 consolidates best practices and removes deprecated flows (implicit, password).

```typescript
// Complete OAuth 2.1 Client
class OAuth21Client {
  private config: OAuthConfig;
  private pkce: PKCEAuthFlow;
  private tokenManager: RefreshTokenManager;
  private stateStore: Map<string, StateData> = new Map();

  constructor(config: OAuthConfig) {
    this.config = config;
    this.pkce = new PKCEAuthFlow();
    this.tokenManager = new RefreshTokenManager(config);
  }

  // Initiate OAuth flow
  async initiateAuthFlow(): Promise<string> {
    const state = this.generateState();
    const stateData: StateData = {
      state,
      createdAt: Date.now(),
      pkceVerifier: this.pkce.getCodeVerifier(),
    };

    this.stateStore.set(state, stateData);

    // Store state in sessionStorage for retrieval after redirect
    sessionStorage.setItem("oauth_state", state);

    return await this.pkce.buildAuthorizationUrl(this.config, state);
  }

  // Handle OAuth callback
  async handleCallback(code: string, state: string): Promise<TokenResponse> {
    // Verify state
    const storedState = sessionStorage.getItem("oauth_state");
    if (state !== storedState) {
      throw new Error("State mismatch - possible CSRF attack");
    }

    const stateData = this.stateStore.get(state);
    if (!stateData) {
      throw new Error("Invalid state");
    }

    // Check state expiry (10 minutes)
    if (Date.now() - stateData.createdAt > 600000) {
      throw new Error("State expired");
    }

    // Exchange code for tokens
    const tokens = await this.pkce.exchangeCodeWithPKCE(this.config, code);

    // Store tokens securely
    this.tokenManager.setTokens(
      tokens.access_token,
      tokens.refresh_token,
      tokens.expires_in,
    );

    // Cleanup
    this.stateStore.delete(state);
    sessionStorage.removeItem("oauth_state");

    return tokens;
  }

  // Make authenticated API request
  async makeAuthenticatedRequest(
    url: string,
    options: RequestInit = {},
  ): Promise<Response> {
    const accessToken = await this.tokenManager.getValidAccessToken();

    const headers = new Headers(options.headers);
    headers.set("Authorization", `Bearer ${accessToken}`);

    return fetch(url, {
      ...options,
      headers,
    });
  }

  // Logout
  async logout(): Promise<void> {
    const accessToken = await this.tokenManager.getValidAccessToken();

    // Revoke tokens if supported
    if (this.config.revocationEndpoint) {
      await this.revokeToken(accessToken);
    }

    this.tokenManager.clearTokens();
  }

  private async revokeToken(token: string): Promise<void> {
    const body = new URLSearchParams({
      token: token,
      token_type_hint: "access_token",
      client_id: this.config.clientId,
      client_secret: this.config.clientSecret,
    });

    await fetch(this.config.revocationEndpoint!, {
      method: "POST",
      headers: {
        "Content-Type": "application/x-www-form-urlencoded",
      },
      body: body.toString(),
    });
  }

  private generateState(): string {
    const array = new Uint8Array(32);
    crypto.getRandomValues(array);
    return Array.from(array, (byte) => byte.toString(16).padStart(2, "0")).join(
      "",
    );
  }
}

interface StateData {
  state: string;
  createdAt: number;
  pkceVerifier: string;
}

interface OAuthConfig extends OAuthConfig {
  revocationEndpoint?: string;
}
```

## Scope Management

```typescript
// OAuth Scope Manager
class ScopeManager {
  private availableScopes: Map<string, ScopeDefinition> = new Map();
  private grantedScopes: Set<string> = new Set();

  constructor() {
    this.initializeScopes();
  }

  private initializeScopes(): void {
    // Define available scopes
    const scopes: ScopeDefinition[] = [
      {
        name: "profile",
        description: "Access to basic profile information",
        required: false,
        sensitive: false,
      },
      {
        name: "email",
        description: "Access to email address",
        required: false,
        sensitive: true,
      },
      {
        name: "read:files",
        description: "Read access to files",
        required: false,
        sensitive: false,
      },
      {
        name: "write:files",
        description: "Write access to files",
        required: false,
        sensitive: true,
      },
      {
        name: "admin",
        description: "Full administrative access",
        required: false,
        sensitive: true,
      },
    ];

    scopes.forEach((scope) => {
      this.availableScopes.set(scope.name, scope);
    });
  }

  requestScopes(scopes: string[]): string[] {
    const validScopes: string[] = [];

    for (const scope of scopes) {
      if (this.availableScopes.has(scope)) {
        validScopes.push(scope);
      }
    }

    return validScopes;
  }

  grantScopes(scopes: string[]): void {
    scopes.forEach((scope) => {
      if (this.availableScopes.has(scope)) {
        this.grantedScopes.add(scope);
      }
    });
  }

  hasScope(scope: string): boolean {
    return this.grantedScopes.has(scope);
  }

  hasAllScopes(scopes: string[]): boolean {
    return scopes.every((scope) => this.hasScope(scope));
  }

  hasAnyScope(scopes: string[]): boolean {
    return scopes.some((scope) => this.hasScope(scope));
  }

  getSensitiveScopes(): string[] {
    return Array.from(this.grantedScopes).filter((scope) => {
      const definition = this.availableScopes.get(scope);
      return definition?.sensitive === true;
    });
  }
}

interface ScopeDefinition {
  name: string;
  description: string;
  required: boolean;
  sensitive: boolean;
}
```

## Token Storage Strategies

```typescript
// Secure Token Storage
class SecureTokenStorage {
  private readonly ACCESS_TOKEN_KEY = "oauth_access_token";
  private readonly REFRESH_TOKEN_KEY = "oauth_refresh_token";
  private readonly EXPIRES_AT_KEY = "oauth_expires_at";

  // Store in memory (most secure for SPAs)
  private memoryStorage: Map<string, string> = new Map();

  // Store tokens in memory only (recommended for SPAs)
  setTokensInMemory(
    accessToken: string,
    refreshToken: string,
    expiresIn: number,
  ): void {
    this.memoryStorage.set(this.ACCESS_TOKEN_KEY, accessToken);
    this.memoryStorage.set(this.REFRESH_TOKEN_KEY, refreshToken);
    this.memoryStorage.set(
      this.EXPIRES_AT_KEY,
      (Date.now() + expiresIn * 1000).toString(),
    );
  }

  getAccessTokenFromMemory(): string | null {
    return this.memoryStorage.get(this.ACCESS_TOKEN_KEY) || null;
  }

  // Store in httpOnly cookies (backend sets)
  // This is the most secure approach for web apps
  async setTokensInCookie(
    accessToken: string,
    refreshToken: string,
    expiresIn: number,
  ): Promise<void> {
    // Send tokens to backend to set httpOnly cookies
    await fetch("/api/auth/set-tokens", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        accessToken,
        refreshToken,
        expiresIn,
      }),
    });
  }

  // Store encrypted in localStorage (less secure, but persistent)
  setTokensInLocalStorage(
    accessToken: string,
    refreshToken: string,
    expiresIn: number,
    encryptionKey: string,
  ): void {
    const encryptedAccess = this.encrypt(accessToken, encryptionKey);
    const encryptedRefresh = this.encrypt(refreshToken, encryptionKey);

    localStorage.setItem(this.ACCESS_TOKEN_KEY, encryptedAccess);
    localStorage.setItem(this.REFRESH_TOKEN_KEY, encryptedRefresh);
    localStorage.setItem(
      this.EXPIRES_AT_KEY,
      (Date.now() + expiresIn * 1000).toString(),
    );
  }

  getAccessTokenFromLocalStorage(encryptionKey: string): string | null {
    const encrypted = localStorage.getItem(this.ACCESS_TOKEN_KEY);
    if (!encrypted) return null;
    return this.decrypt(encrypted, encryptionKey);
  }

  // Simple XOR encryption (use proper crypto library in production)
  private encrypt(text: string, key: string): string {
    return btoa(
      text
        .split("")
        .map((char, i) =>
          String.fromCharCode(
            char.charCodeAt(0) ^ key.charCodeAt(i % key.length),
          ),
        )
        .join(""),
    );
  }

  private decrypt(encoded: string, key: string): string {
    return atob(encoded)
      .split("")
      .map((char, i) =>
        String.fromCharCode(
          char.charCodeAt(0) ^ key.charCodeAt(i % key.length),
        ),
      )
      .join("");
  }

  clearAllTokens(): void {
    this.memoryStorage.clear();
    localStorage.removeItem(this.ACCESS_TOKEN_KEY);
    localStorage.removeItem(this.REFRESH_TOKEN_KEY);
    localStorage.removeItem(this.EXPIRES_AT_KEY);
  }
}
```

## OAuth Provider Integration Examples

```typescript
// Google OAuth Integration
class GoogleOAuthProvider {
  private client: OAuth21Client;

  constructor(clientId: string, clientSecret: string) {
    const config: OAuthConfig = {
      clientId,
      clientSecret,
      authorizationEndpoint: "https://accounts.google.com/o/oauth2/v2/auth",
      tokenEndpoint: "https://oauth2.googleapis.com/token",
      redirectUri: "http://localhost:3000/callback",
      scope: ["openid", "profile", "email"],
    };

    this.client = new OAuth21Client(config);
  }

  async login(): Promise<void> {
    const authUrl = await this.client.initiateAuthFlow();
    window.location.href = authUrl;
  }

  async handleCallback(code: string, state: string): Promise<UserProfile> {
    const tokens = await this.client.handleCallback(code, state);
    return await this.getUserProfile(tokens.access_token);
  }

  private async getUserProfile(accessToken: string): Promise<UserProfile> {
    const response = await fetch(
      "https://www.googleapis.com/oauth2/v2/userinfo",
      {
        headers: {
          Authorization: `Bearer ${accessToken}`,
        },
      },
    );

    return await response.json();
  }
}

// GitHub OAuth Integration
class GitHubOAuthProvider {
  private client: OAuth21Client;

  constructor(clientId: string, clientSecret: string) {
    const config: OAuthConfig = {
      clientId,
      clientSecret,
      authorizationEndpoint: "https://github.com/login/oauth/authorize",
      tokenEndpoint: "https://github.com/login/oauth/access_token",
      redirectUri: "http://localhost:3000/callback",
      scope: ["user", "repo"],
    };

    this.client = new OAuth21Client(config);
  }

  async login(): Promise<void> {
    const authUrl = await this.client.initiateAuthFlow();
    window.location.href = authUrl;
  }

  async getRepositories(accessToken: string): Promise<Repository[]> {
    const response = await fetch("https://api.github.com/user/repos", {
      headers: {
        Authorization: `Bearer ${accessToken}`,
        Accept: "application/vnd.github.v3+json",
      },
    });

    return await response.json();
  }
}

interface UserProfile {
  id: string;
  email: string;
  name: string;
  picture?: string;
}

interface Repository {
  id: number;
  name: string;
  full_name: string;
  private: boolean;
}
```

## Real-World OAuth Implementation (React)

```typescript
// React OAuth Hook
import { useState, useEffect, useCallback } from 'react';

interface UseOAuthResult {
  isAuthenticated: boolean;
  isLoading: boolean;
  error: Error | null;
  login: () => Promise<void>;
  logout: () => Promise<void>;
  makeAuthRequest: <T>(url: string, options?: RequestInit) => Promise<T>;
}

function useOAuth(config: OAuthConfig): UseOAuthResult {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  const [client] = useState(() => new OAuth21Client(config));

  // Check authentication status on mount
  useEffect(() => {
    const checkAuth = async () => {
      try {
        // Try to get a valid access token
        const token = await client.tokenManager.getValidAccessToken();
        setIsAuthenticated(!!token);
      } catch (err) {
        setIsAuthenticated(false);
      } finally {
        setIsLoading(false);
      }
    };

    checkAuth();
  }, [client]);

  // Handle OAuth callback
  useEffect(() => {
    const handleCallback = async () => {
      const params = new URLSearchParams(window.location.search);
      const code = params.get('code');
      const state = params.get('state');

      if (code && state) {
        try {
          setIsLoading(true);
          await client.handleCallback(code, state);
          setIsAuthenticated(true);

          // Clean URL
          window.history.replaceState({}, document.title, window.location.pathname);
        } catch (err) {
          setError(err as Error);
        } finally {
          setIsLoading(false);
        }
      }
    };

    handleCallback();
  }, [client]);

  const login = useCallback(async () => {
    try {
      setIsLoading(true);
      const authUrl = await client.initiateAuthFlow();
      window.location.href = authUrl;
    } catch (err) {
      setError(err as Error);
      setIsLoading(false);
    }
  }, [client]);

  const logout = useCallback(async () => {
    try {
      setIsLoading(true);
      await client.logout();
      setIsAuthenticated(false);
    } catch (err) {
      setError(err as Error);
    } finally {
      setIsLoading(false);
    }
  }, [client]);

  const makeAuthRequest = useCallback(
    async <T,>(url: string, options?: RequestInit): Promise<T> => {
      const response = await client.makeAuthenticatedRequest(url, options);

      if (!response.ok) {
        throw new Error(`Request failed: ${response.statusText}`);
      }

      return await response.json();
    },
    [client]
  );

  return {
    isAuthenticated,
    isLoading,
    error,
    login,
    logout,
    makeAuthRequest,
  };
}

// Usage in React Component
function App() {
  const config: OAuthConfig = {
    clientId: 'your-client-id',
    clientSecret: 'your-client-secret',
    authorizationEndpoint: 'https://provider.com/oauth/authorize',
    tokenEndpoint: 'https://provider.com/oauth/token',
    redirectUri: 'http://localhost:3000/callback',
    scope: ['profile', 'email'],
  };

  const { isAuthenticated, isLoading, login, logout, makeAuthRequest } = useOAuth(config);

  if (isLoading) {
    return <div>Loading...</div>;
  }

  return (
    <div>
      {isAuthenticated ? (
        <>
          <button onClick={logout}>Logout</button>
          <UserProfile makeAuthRequest={makeAuthRequest} />
        </>
      ) : (
        <button onClick={login}>Login with OAuth</button>
      )}
    </div>
  );
}
```

## Production Security Best Practices

```typescript
// Security Hardening Layer
class OAuthSecurityLayer {
  private static readonly MAX_STATE_AGE = 600000; // 10 minutes
  private static readonly MAX_TOKEN_AGE = 3600000; // 1 hour

  // Validate redirect URI to prevent open redirects
  static validateRedirectUri(
    redirectUri: string,
    allowedUris: string[],
  ): boolean {
    try {
      const url = new URL(redirectUri);

      // Must be exact match, no wildcards
      return allowedUris.some((allowed) => {
        const allowedUrl = new URL(allowed);
        return (
          url.protocol === allowedUrl.protocol &&
          url.hostname === allowedUrl.hostname &&
          url.port === allowedUrl.port &&
          url.pathname === allowedUrl.pathname
        );
      });
    } catch {
      return false;
    }
  }

  // Implement rate limiting for token requests
  static createRateLimiter() {
    const requests = new Map<string, number[]>();
    const MAX_REQUESTS = 10;
    const WINDOW_MS = 60000; // 1 minute

    return (clientId: string): boolean => {
      const now = Date.now();
      const timestamps = requests.get(clientId) || [];

      // Remove old timestamps
      const recentTimestamps = timestamps.filter((ts) => now - ts < WINDOW_MS);

      if (recentTimestamps.length >= MAX_REQUESTS) {
        return false; // Rate limit exceeded
      }

      recentTimestamps.push(now);
      requests.set(clientId, recentTimestamps);
      return true;
    };
  }

  // Validate JWT token (if using JWT access tokens)
  static async validateJWTToken(
    token: string,
    publicKey: string,
  ): Promise<TokenPayload> {
    const [headerB64, payloadB64, signature] = token.split(".");

    // Verify signature
    const data = `${headerB64}.${payloadB64}`;
    const isValid = await this.verifySignature(data, signature, publicKey);

    if (!isValid) {
      throw new Error("Invalid token signature");
    }

    // Decode payload
    const payload = JSON.parse(atob(payloadB64));

    // Validate expiration
    if (payload.exp && Date.now() >= payload.exp * 1000) {
      throw new Error("Token expired");
    }

    // Validate not before
    if (payload.nbf && Date.now() < payload.nbf * 1000) {
      throw new Error("Token not yet valid");
    }

    return payload;
  }

  private static async verifySignature(
    data: string,
    signature: string,
    publicKey: string,
  ): Promise<boolean> {
    // Implement proper signature verification using Web Crypto API
    // This is a simplified example
    return true;
  }

  // Implement token introspection
  static async introspectToken(
    token: string,
    introspectionEndpoint: string,
    clientId: string,
    clientSecret: string,
  ): Promise<TokenIntrospection> {
    const credentials = btoa(`${clientId}:${clientSecret}`);

    const response = await fetch(introspectionEndpoint, {
      method: "POST",
      headers: {
        "Content-Type": "application/x-www-form-urlencoded",
        Authorization: `Basic ${credentials}`,
      },
      body: new URLSearchParams({ token }),
    });

    return await response.json();
  }
}

interface TokenPayload {
  iss: string; // Issuer
  sub: string; // Subject
  aud: string; // Audience
  exp: number; // Expiration
  nbf: number; // Not before
  iat: number; // Issued at
  scope: string;
}

interface TokenIntrospection {
  active: boolean;
  scope?: string;
  client_id?: string;
  username?: string;
  exp?: number;
}
```

## Best Practices

### 1. **Always Use PKCE**

```typescript
// Even for confidential clients, PKCE adds security
const pkce = new PKCEAuthFlow();
```

### 2. **Never Store Tokens in localStorage**

```typescript
// Prefer memory storage or httpOnly cookies
const storage = new SecureTokenStorage();
storage.setTokensInMemory(accessToken, refreshToken, expiresIn);
```

### 3. **Implement Token Rotation**

```typescript
// Rotate refresh tokens on each use
async refreshAccessToken() {
  // Old refresh token becomes invalid
  // New refresh token is issued
}
```

### 4. **Use Short-Lived Access Tokens**

```typescript
// 15-60 minutes maximum
const expiresIn = 900; // 15 minutes
```

### 5. **Validate All Inputs**

```typescript
// Validate redirect URIs, state, scopes
OAuthSecurityLayer.validateRedirectUri(uri, allowedUris);
```

### 6. **Implement Rate Limiting**

```typescript
const rateLimiter = OAuthSecurityLayer.createRateLimiter();
if (!rateLimiter(clientId)) {
  throw new Error("Rate limit exceeded");
}
```

### 7. **Use State Parameter**

```typescript
// Prevent CSRF attacks
const state = generateRandomState();
sessionStorage.setItem("oauth_state", state);
```

### 8. **Secure Token Transmission**

```typescript
// Always use HTTPS
// Never log or expose tokens
console.log("Token received"); // Good
console.log(token); // BAD!
```

### 9. **Implement Proper Error Handling**

```typescript
try {
  await client.handleCallback(code, state);
} catch (error) {
  if (error.message.includes("State mismatch")) {
    // Possible CSRF attack
    logSecurityEvent("CSRF_DETECTED", { error });
  }
}
```

### 10. **Regular Security Audits**

```typescript
// Monitor for:
// - Unusual token requests
// - Failed authentication attempts
// - Token introspection failures
```

## Key Takeaways

1. **OAuth 2.1 is the current standard** - consolidates best practices and removes deprecated grant types (implicit, password)

2. **PKCE is mandatory for public clients** - SPAs and mobile apps must use PKCE to prevent authorization code interception

3. **Authorization Code Flow is most secure** - use with PKCE for web apps, always prefer over deprecated implicit flow

4. **Token storage is critical** - memory storage > httpOnly cookies > encrypted localStorage, never plain localStorage

5. **State parameter prevents CSRF** - always generate cryptographically random state and verify on callback

6. **Refresh tokens enable seamless UX** - obtain new access tokens without re-authentication, implement rotation

7. **Scopes define permissions** - request minimum necessary scopes, understand sensitive vs non-sensitive scopes

8. **Client credentials for M2M** - server-to-server communication without user interaction, use short-lived tokens

9. **Security layers are essential** - rate limiting, redirect URI validation, token introspection, audit logging

10. **Production requires HTTPS** - OAuth flows must use TLS/SSL, validate certificates, secure token transmission end-to-end
