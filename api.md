# API Documentation

Base URL: `https://api.zikrou.com/api/v1`

All endpoints return JSON. Authenticated endpoints require an `Authorization: Bearer <token>` header.

---

## Authentication

### Register

```
POST /auth/register
```

**Body**
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response** `200 OK`
```json
{
  "accessToken": "eyJhbGci...",
  "refreshToken": "uuid-v4",
  "userId": 1,
  "email": "user@example.com",
  "role": "USER"
}
```

---

### Login

```
POST /auth/login
```

**Body**
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response** `200 OK` — same as register.

---

### Refresh Token

```
POST /auth/refresh
```

**Body**
```json
{
  "refreshToken": "uuid-v4"
}
```

**Response** `200 OK`
```json
{
  "accessToken": "eyJhbGci...",
  "refreshToken": "new-uuid-v4"
}
```

---

### Logout

```
POST /auth/logout
```

Revokes the refresh token. The access token expires naturally after 15 minutes.

---

## Content (public — no auth required for GET)

### Zikrs

```
GET  /zikr                          List (paginated, filterable)
GET  /zikr/{id}                     Get by ID
GET  /zikr/search?q=baye            Full-text search
GET  /zikr/album/{albumId}          By album
GET  /zikr/daira/{dairaId}          By daira
GET  /zikr/{id}/stats               Play count, likes, downloads
POST /zikr/{id}/play                Record a play (authenticated)
POST /zikr/{id}/ratings             Rate 1–5 (authenticated)
GET  /zikr/{id}/ratings/me          My rating (authenticated)
GET  /zikr/{id}/comments            Comments (public)
POST /zikr/{id}/comments            Add comment (authenticated)
```

**Zikr list filters**

| Parameter | Type | Description |
|---|---|---|
| `page` | int | Page number (default: 0) |
| `size` | int | Page size (default: 20) |
| `genreId` | long | Filter by genre |
| `dairaId` | long | Filter by daira |
| `zakirId` | long | Filter by zakir |
| `albumId` | long | Filter by album |
| `countryId` | long | Filter by country |
| `isPremium` | boolean | Filter premium content |
| `language` | string | Filter by language code |
| `sortBy` | string | Field to sort by |
| `sortDir` | string | `asc` or `desc` |

---

### Zakirs

```
GET  /zakir                         List (paginated)
GET  /zakir/{id}                    Get by ID
POST /zakir/{id}/follow             Follow (authenticated)
DELETE /zakir/{id}/follow           Unfollow (authenticated)
```

---

### Dairas

```
GET  /daira                         List (paginated)
GET  /daira/{id}                    Get by ID
POST /daira/{id}/follow             Follow (authenticated)
DELETE /daira/{id}/follow           Unfollow (authenticated)
```

---

### Albums

```
GET  /album                         List
GET  /album/{id}                    Get by ID
GET  /album/zakir/{zakirId}         By Zakir
```

---

### Genres, Tags, Countries

```
GET  /genre                         List all genres
GET  /tag                           List all tags
GET  /country                       List all countries
```

---

## User (authenticated)

```
GET  /users/me                      My profile
PUT  /users/me                      Update profile
PUT  /users/me/password             Change password
GET  /users/me/settings             My settings
PUT  /users/me/settings             Update settings
```

---

## Favorites (authenticated)

```
GET    /favorites                   My favorites
POST   /favorites/{zikrId}          Add to favorites
DELETE /favorites/{zikrId}          Remove from favorites
```

---

## Listening History (authenticated)

```
GET  /history                       My listening history
POST /history                       Record or update progress
```

---

## Playlists (authenticated)

```
GET    /playlists                   My playlists
GET    /playlists/{id}              Get with tracks
POST   /playlists                   Create
PUT    /playlists/{id}              Update
DELETE /playlists/{id}              Delete
POST   /playlists/{id}/zikr/{zikrId}   Add track
DELETE /playlists/{id}/zikr/{zikrId}   Remove track
PUT    /playlists/{id}/reorder      Reorder tracks
```

---

## Downloads (authenticated — premium)

```
POST   /downloads/{zikrId}          Download a Zikr
GET    /downloads                   My downloads
DELETE /downloads/{id}              Remove download
```

---

## Search (public)

```
GET  /search?q=baye&type=ALL        Search across Zikrs, Zakirs, Dairas
GET  /search/history                My search history (authenticated)
DELETE /search/history              Clear history (authenticated)
```

Search types: `ALL`, `ZIKR`, `ZAKIR`, `DAIRA`

---

## Home (authenticated)

```
GET  /home                          Home feed
```

**Response**
```json
{
  "trending": [...],
  "recentlyAdded": [...],
  "continueListening": [...],
  "fromFollowedZakirs": [...],
  "fromFollowedDairas": [...]
}
```

---

## Community (authenticated)

```
POST   /zikr/{id}/comments          Add comment
DELETE /comments/{id}               Delete own comment
PUT    /comments/{id}               Edit own comment
POST   /reports                     Report content
POST   /suggestions                 Submit content suggestion
GET    /suggestions/mine            My suggestions
```

---

## Subscriptions & Payments (authenticated)

```
GET  /subscription/me               My current subscription
POST /subscription                  Subscribe to a plan
GET  /payments                      My payment history
POST /payments                      Record a payment
```

---

## Public Configuration

```
GET  /config                        App configuration (min_app_build, etc.)
```

---

## Admin (ROLE_ADMIN only)

```
GET  /admin/users                   All users
PUT  /admin/users/{id}/status       Block or activate user
GET  /admin/reports                 Pending reports
PUT  /admin/reports/{id}            Handle report
GET  /admin/suggestions             All suggestions
PUT  /admin/suggestions/{id}        Handle suggestion
PUT  /admin/config/{key}            Update app config
```

---

## HTTP Status Codes

| Code | Meaning |
|---|---|
| 200 | Success |
| 201 | Created |
| 204 | No content (delete) |
| 400 | Bad request / validation error |
| 401 | Unauthorized |
| 403 | Forbidden (wrong role) |
| 404 | Not found |
| 409 | Conflict (duplicate) |
| 500 | Internal server error |
