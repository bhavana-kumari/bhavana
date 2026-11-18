# NestFind

NestFind connects owners who have an independent house, apartment or flat, or complete floor with tenants looking for accommodation. Tenants search by city, locality, current location, property type, and furnishing. Owners publish listings with privacy details, amenities, photos, and a map pin.

The app is a React (Vite) frontend and an Express API backed by MongoDB. Location search uses GeoJSON and a `2dsphere` index.

## Run locally

Requirements: Node.js 18 or newer, and a MongoDB instance (local or Atlas).

```bash
cp backend/.env.example backend/.env
```

On Windows PowerShell:

```powershell
Copy-Item backend\.env.example backend\.env
```

Set these in `backend/.env` before starting:

- `MONGODB_URI` — local example `mongodb://127.0.0.1:27017/nestfind`, or a MongoDB Atlas connection string.
- `JWT_SECRET` — a long random string. Generate one with `node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"`.

Then:

```bash
npm run install:all
npm run dev
```

- API: http://localhost:5000
- Web: http://localhost:5173

Vite proxies `/api` and `/uploads` to the API, so the frontend can use relative URLs. `frontend/.env.example` sets `VITE_API_URL=/api`. Copy it to `frontend/.env` only if you need to point the browser at a different API origin. In that case set `CLIENT_URL` to the frontend origin and, for a cross-site deployment over HTTPS, set `COOKIE_SAMESITE=none`.

Production-style checks:

```bash
npm test
npm run build
```

`npm start` runs the API only. Serve `frontend/dist` separately and proxy `/api` and `/uploads` to it, or set `VITE_API_URL` at build time.

## Authentication

Passwords are hashed with bcryptjs and are never returned by the API.

On register and login the API signs a JWT and stores it in an **httpOnly cookie** named `token`. The cookie is not readable by JavaScript, so the token is not kept in `localStorage`. Axios sends it with `withCredentials: true`. Logout clears the cookie. A `Bearer` token is also accepted for non-browser clients, but the web app uses the cookie.

The cookie is `SameSite=Lax` and is `Secure` in production. Invalid or expired tokens return 401.

Roles are `tenant` and `owner`, chosen at registration. Listing changes require the owner role and ownership of that listing. Favorites and new enquiries require the tenant role.

## Images

Uploads accept JPEG, PNG, and WebP, up to 8 images and 5 MB each.

Cloudinary is optional. If `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, and `CLOUDINARY_API_SECRET` are all set, photos are stored there. If any of them is blank, photos are saved on disk under `backend/uploads` and served at `/uploads`. The rest of the app works without Cloudinary. `GET /api/health` reports `uploadProvider` as `cloudinary` or `local`.

## Maps and geocoding

Maps use Leaflet and OpenStreetMap tiles. Property coordinates are stored as GeoJSON points (`[longitude, latitude]`).

Place search and reverse geocoding use Nominatim by default (`GEOCODING_PROVIDER=nominatim`). No API key is required. Set `GEOCODING_USER_AGENT` to a real contact string before relying on search; Nominatim’s usage policy asks for one. Radius search uses MongoDB `$geoNear` and `$maxDistance`. Public responses round the pin to about 100 metres unless the viewer owns the listing or the owner turned on “share the exact location”.

## Sample data

```bash
npm run seed
```

This creates labelled development data and does not invent listings in the UI. The seed replaces only these two accounts and the listings they own:

| Account | Email | Password |
| --- | --- | --- |
| Owner | owner@nestfind.local | Password123 |
| Tenant | tenant@nestfind.local | Password123 |

Sample listings are stored with `sample: true`, say so in the description, and show a Sample badge. An empty database shows an empty state on the homepage instead of placeholder homes.

## What needs which service

| Feature | Required |
| --- | --- |
| Accounts, listings, search, enquiries, favorites | MongoDB and `JWT_SECRET` |
| City and map search | Nominatim (or the configured geocoding URL) reachable from the API |
| Photos | Local disk, or Cloudinary if those three variables are set |
| Current location | Browser permission; the app still works if it is denied |

There is no payment flow, recommendation model, or realtime chat.

## API

Responses use `{ success, data }` or `{ success: false, message, errors? }`.

Auth

- `POST /api/auth/register`
- `POST /api/auth/login`
- `POST /api/auth/logout`
- `GET /api/auth/me`

Properties

- `GET /api/properties` — query: `location`, `lat`, `lng`, `radius` (km, 1–50), `propertyType`, `furnishing`, `page`, `limit`, `sort` (`newest`, `rent_asc`, `rent_desc`, `distance`)
- `GET /api/properties/:id`
- `POST /api/properties` — owner
- `PUT /api/properties/:id` — owner of that listing
- `DELETE /api/properties/:id` — owner of that listing
- `GET /api/properties/my-listings` — owner
- `PATCH /api/properties/:id/status` — body `{ availabilityStatus: "available" | "unavailable" }`

Enquiries

- `POST /api/enquiries` — tenant, body `{ propertyId, message }`
- `GET /api/enquiries/my-enquiries` — tenant
- `GET /api/enquiries/received` — owner
- `PATCH /api/enquiries/:id/status` — body `{ status: "pending" | "responded" | "closed" }`. A tenant can only close their own enquiry.

A second enquiry for the same home is rejected while one is still pending, or within 24 hours of the last one.

Favorites (tenant)

- `GET /api/favorites`
- `POST /api/favorites/:propertyId`
- `DELETE /api/favorites/:propertyId`

Users

- `GET /api/users/profile`
- `PUT /api/users/profile` — body `{ name, phone }`

Other

- `POST /api/uploads` — owner, multipart field `photos`
- `GET /api/geo/search?q=`
- `GET /api/geo/reverse?lat=&lng=`
- `GET /api/health`

Property types are `independent_house`, `apartment`, and `complete_floor`. Furnishing is `furnished` or `unfurnished`.
