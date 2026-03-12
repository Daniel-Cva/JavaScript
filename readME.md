# Core Platform Logic & Dummy Code Reference

This document outlines the algorithms, technologies, and dummy code for the key features requested: Founder Networking, Product Fetching, Request Routing, and Authentication.

---

## 1. Founder Networking (Live Location within 500m)

### Technology & Algorithm
* **Database (Cloudflare D1):** SQLite does not have native spatial indexing (like PostGIS). Running complex math on every row is inefficient.
* **Algorithm (Bounding Box + Haversine):**
    1.  **Phase 1 (SQL):** We calculate a crude "Bounding Box" (a square) around the user's coordinates. 500 meters is roughly `0.0045` degrees of latitude. We query D1 to only return founders within `lat +/- 0.0045` and `lng +/- (0.0045 / cos(lat))`. This uses standard SQL indexes and is extremely fast.
    2.  **Phase 2 (JS Worker):** We use the **Haversine Formula** in JavaScript to filter the bounding box results into a perfect 500m circle.
* **Realtime:** Client-side polling (every 10s) or WebSockets via Cloudflare Durable Objects.

### Dummy Code (Cloudflare Worker API - Node/JS)

```javascript
// util/distance.js - Haversine Formula
export function getDistanceInMeters(lat1, lon1, lat2, lon2) {
    const R = 6371e3; // Earth radius in meters
    const rad = Math.PI / 180;
    const dLat = (lat2 - lat1) * rad;
    const dLon = (lon2 - lon1) * rad;
    const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
              Math.cos(lat1 * rad) * Math.cos(lat2 * rad) *
              Math.sin(dLon / 2) * Math.sin(dLon / 2);
    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    return R * c;
}

// Fetch Nearby Founders API (+server.js)
export async function GET({ url, platform }) {
    const lat = parseFloat(url.searchParams.get('lat'));
    const lng = parseFloat(url.searchParams.get('lng'));
    
    // 500m Bounding Box (~0.0045 degrees)
    const latOffset = 0.0045;
    const lngOffset = 0.0045 / Math.cos(lat * (Math.PI / 180));

    // Fast SQL query using bounding box (indexes on lat/lng recommended)
    const { results } = await platform.env.DB.prepare(`
        SELECT f.founder_id, f.name, b.lat, b.long 
        FROM business_founders f
        JOIN biz_data b ON f.biz_id = b.id
        WHERE b.lat BETWEEN ? AND ? 
        AND b.long BETWEEN ? AND ?
    `).bind(lat - latOffset, lat + latOffset, lng - lngOffset, lng + lngOffset).all();

    // Precise 500m filter in JS
    const nearbyFounders = results.filter(f => 
        getDistanceInMeters(lat, lng, f.lat, f.long) <= 500
    );

    return new Response(JSON.stringify(nearbyFounders));
}
```

### Connection Request (Yes/No Handshake)

```svelte
<!-- FounderMap.svelte -->
<script>
    export let nearbyFounders = [];
    
    async function sendConnectRequest(founderId) {
        await fetch('/api/founder/connect', {
            method: 'POST',
            body: JSON.stringify({ targetFounderId: founderId })
        });
        alert("Connection request sent!");
    }
</script>

<div class="p-4 grid grid-cols-1 gap-4">
    {#each nearbyFounders as founder}
        <div class="border rounded-lg p-4 shadow-sm flex justify-between items-center">
            <h3 class="font-bold">{founder.name}</h3>
            <p class="text-sm text-gray-500">{(founder.distance).toFixed(0)}m away</p>
            <button 
                on:click={() => sendConnectRequest(founder.founder_id)}
                class="bg-blue-600 text-white px-4 py-2 rounded-md hover:bg-blue-700">
                Connect
            </button>
        </div>
    {/each}
</div>
```
*(When requested, the target founder gets a notification triggering an endpoint `POST /api/founder/respond` with `{ status: 'yes' | 'no' }`)*

---

## 2. Product Fetching API Data

Because your schema uses dynamic tables (`biz_{biz_id}_items`), fetching products requires knowing the business ID, or querying a specific business layout.

```javascript
// src/routes/api/business/[bid]/products/+server.js
export async function GET({ params, platform }) {
    const bizId = params.bid;
    // Prevent SQL injection by validating bizId is numeric
    if (!/^\d+$/.test(bizId)) return new Response("Invalid ID", { status: 400 });

    const tableName = `biz_${bizId}_items`;
    
    // Check if table exists (or wrap in try/catch)
    try {
        const { results } = await platform.env.DB.prepare(`
            SELECT * FROM ${tableName} ORDER BY created_at DESC
        `).all();
        
        return new Response(JSON.stringify(results), { status: 200 });
    } catch(e) {
        // Table doesn't exist yet (business hasn't added items)
        return new Response(JSON.stringify([]), { status: 200 });
    }
}
```

---

## 3. User Requirements (Matching Nearby Sellers)

**Logic:** When a user creates a request with a specific category (e.g., Subcategory 3 - "Plumbing"), we need to find businesses registered under that subcategory near the user's location.

1. User sends Request (lat, lng, subcat_id).
2. Query `sub_{subcat_id}_biz_{biz_id}` template tables. (Wait, your schema separates finding the businesses into `sub_{subcat_id}_biz_{biz_id}`. A better query is to track a master mapping table or query `biz_data` joined with `sub_categories` if possible). Let's assume you have a `business_subcategories` mapping table, OR we query the dynamic tables.

```javascript
// src/routes/api/requests/create/+server.js
export async function POST({ request, platform }) {
    const data = await request.json(); // { lat, lng, subcat_id, details }
    
    // 1. Save user's request globally or in user_{user_id}_request
    
    // 2. Find nearby businesses in this category
    // In D1, we might need a mapping table mapping subcategories to businesses 
    // to find them efficiently without knowing the biz_id beforehand.
    const { results: businesses } = await platform.env.DB.prepare(`
        SELECT b.id, b.lat, b.long, b.bemail 
        FROM biz_data b
        JOIN biz_subcat_mapping m ON b.id = m.biz_id
        WHERE m.subcat_id = ?
    `).bind(data.subcat_id).all();

    // 3. Filter distance using Bounding Box array / Haversine (like above)
    const radiusMeters = 5000; // 5km search radius for requests
    const nearbySellers = businesses.filter(b => 
        getDistanceInMeters(data.lat, data.lng, b.lat, b.long) <= radiusMeters
    );

    // 4. Insert request into EACH nearby seller's dynamic table
    for (const seller of nearbySellers) {
        // Notify seller via biz_{biz_id}_request table
        await platform.env.DB.prepare(`
            INSERT INTO biz_${seller.id}_request (sub_category, user_id, lat, long, request_id)
            VALUES (?, ?, ?, ?, ?)
        `).bind(data.subcategory_name, data.user_id, data.lat, data.lng, data.request_id).run();
    }

    return new Response(JSON.stringify({ matchedSellersCount: nearbySellers.length }));
}
```

---

## 4. Authentication (Bcrypt, JWT with 2h Expiry)

### Technology
* **Cloudflare Workers constraints:** Standard Node native C++ `bcrypt` will NOT work on Cloudflare Workers. You must use `bcryptjs` (pure JS) or WebCrypto API (`PBKDF2` / `Scrypt`).
* **JWT:** Use the `jose` library (lightning fast edge-compatible JWT library).

### Registration & Login (Dummy Auth Setup)
```javascript
// src/routes/api/auth/login/+server.js
import bcrypt from 'bcryptjs';
import * as jose from 'jose';

const JWT_SECRET = new TextEncoder().encode("SUPER_SECRET_KEY_CHANGE_ME");

export async function POST({ request, platform, cookies }) {
    const { email, password, role } = await request.json(); // role: 'user' | 'biz' | 'admin'
    
    // Determine table based on role
    const table = role === 'biz' ? 'biz_login' : role === 'admin' ? 'sa_login' : 'user_login';
    
    const user = await platform.env.DB.prepare(`SELECT * FROM ${table} WHERE email = ?`)
                                      .bind(email).first();
                                      
    if (!user || !(await bcrypt.compare(password, user.password_hash))) {
        return new Response("Invalid credentials", { status: 401 });
    }

    // Generate JWT (Expires in 2 hours)
    const jwt = await new jose.SignJWT({ id: user.id, email: user.email, role })
        .setProtectedHeader({ alg: 'HS256' })
        .setIssuedAt()
        .setExpirationTime('2h') // 2 hours expiry
        .sign(JWT_SECRET);

    // Set HTTP-Only Cookie
    cookies.set('session_token', jwt, {
        path: '/',
        httpOnly: true,
        secure: true,
        sameSite: 'strict',
        maxAge: 60 * 60 * 2 // 2 hours in seconds
    });

    // Role-based Redirect Logic (Client or Server handles this)
    let redirectUrl = '/';
    if (role === 'biz') redirectUrl = '/business-dashboard';
    else if (role === 'admin') redirectUrl = '/admin-dashboard';
    else redirectUrl = '/user-home';

    return new Response(JSON.stringify({ redirectUrl }), { status: 200 });
}
```

### Auto-Logout (Expiry Check Hook)
In SvelteKit, `hooks.server.js` acts as middleware on every single page load.

```javascript
// src/hooks.server.js
import * as jose from 'jose';

const JWT_SECRET = new TextEncoder().encode("SUPER_SECRET_KEY_CHANGE_ME");

export async function handle({ event, resolve }) {
    const token = event.cookies.get('session_token');

    if (token) {
        try {
            // Verify token. If expired, this throws an error.
            const { payload } = await jose.jwtVerify(token, JWT_SECRET);
            event.locals.user = payload; // accessible in any route
            
        } catch (err) {
            // Token is invalid or EXPERIED (> 2 hours)
            // Force logout by deleting the cookie
            event.cookies.delete('session_token', { path: '/' });
            event.locals.user = null;
            
            // Redirect to login if they are on a protected route
            const path = event.url.pathname;
            if (path.startsWith('/admin') || path.startsWith('/business') || path.startsWith('/user-home')) {
                return new Response('Redirect', { status: 303, headers: { Location: '/login' } });
            }
        }
    }

    return resolve(event);
}
```

---

## 5. Advanced Spatial Algorithms & Logic

### Plane Geometry (Euclidean Approximation)
*For hyper-local searches (2km - 5km). Faster than Haversine, reducing CPU load on edge workers by treating the local area as a flat plane.*

```javascript
// util/distance.js - Flat-Earth Approximation (Good for < 5km)
export function getDistancePlaneGeometry(lat1, lon1, lat2, lon2) {
    const R = 6371e3; // Earth radius in meters
    const rad = Math.PI / 180;
    
    // Convert to radians
    const lat1Rad = lat1 * rad;
    const lat2Rad = lat2 * rad;
    const lon1Rad = lon1 * rad;
    const lon2Rad = lon2 * rad;
    
    // Pythagorean theorem on equirectangular projection
    const x = (lon2Rad - lon1Rad) * Math.cos((lat1Rad + lat2Rad) / 2);
    const y = (lat2Rad - lat1Rad);
    
    return Math.sqrt(x * x + y * y) * R;
}
```

### Gravity Law of Spatial Interaction
*Determines the matching probability ($P_{ui}$) between a user and a provider based on distance ($d_{ui}$) and the provider's Success Score ($S_i$).*

```javascript
// util/matching.js - Gravity Law Probability Model
export function calculateMatchProbability(successScore, distanceInMeters) {
    // Avoid division by zero, set minimum distance to 1 meter
    const safeDistance = Math.max(distanceInMeters, 1);
    
    // Formula: P_ui = S_i / (d_ui^2)
    const probability = successScore / Math.pow(safeDistance, 2);
    
    // Note: You can normalize or scale the probability score 
    // depending on the typical 'successScore' ranges in your DB.
    return probability;
}
```

### K-Area Obfuscation (Differential Privacy)
*Adds mathematical noise to live coordinates to protect founder privacy during commutes, ensuring they remain "nearby" without exposing exact home/warehouse locations.*

```javascript
// util/privacy.js - K-Area Disclosure Control
export function obfuscateLocation(lat, lng, maxOffsetMeters = 200) {
    const R = 6371e3; // Earth radius in meters
    
    // Generate random offset within the designated privacy radius
    const offsetMeters = Math.random() * maxOffsetMeters;
    
    // Generate random direction (angle)
    const angle = Math.random() * 2 * Math.PI;
    
    // Calculate new coordinate offsets
    const latOffset = (offsetMeters * Math.cos(angle)) / R;
    const lngOffset = (offsetMeters * Math.sin(angle)) / (R * Math.cos(lat * Math.PI / 180));
    
    // Convert back from radians to degrees
    return {
        lat: lat + (latOffset * 180 / Math.PI),
        lng: lng + (lngOffset * 180 / Math.PI)
    };
}
```
