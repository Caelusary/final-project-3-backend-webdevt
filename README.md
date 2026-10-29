# Loadout (backend)

The REST API for Loadout, a marketplace for computer peripherals: independent shops list keyboards, mice, headsets, webcams, mousepads and accessories, customers buy from several shops in one checkout, and a platform admin approves sellers and moderates listings. Built with MongoDB, Express, React and Node (Group 3, C3A final project).

This repo is the Express API. The React client lives in [final-project-3-frontend-webdevt](https://github.com/Caelusary/final-project-3-frontend-webdevt).

- API health check: https://loadout-api-lk00.onrender.com/api/health
- Live site: https://loadout-brown.vercel.app

In production the API only answers requests that come through the site (Vercel forwards `/api` with a shared `PROXY_SECRET`), so apart from the health check, open it through the live site.

## Running locally

Requires Node 20.19 or newer. No database to install: when `MONGODB_URI` is empty, the API starts an in-memory MongoDB replica set and seeds it (the first run downloads a MongoDB binary, about 780 MB).

```bash
npm install
```

```bash
npm run dev
```

The API runs on http://localhost:5000. Data resets when it stops. Start the frontend repo beside it to use the site.

To use a real database, copy `.env.example` to `.env`, set `MONGODB_URI` to an Atlas connection string, then seed it. Seeding deletes everything in that database first, so for anything that isn't on this machine it asks for an explicit flag:

```bash
npm run seed -- --wipe-remote
```

Seed a live database once, then take its connection string out of your local `.env` so nobody reseeds it by accident. Every variable is described in [.env.example](.env.example).

## Services

| Service | Used for |
|---|---|
| Render | Hosts the API ([render.yaml](render.yaml)), with `/api/health` (which fails when MongoDB is disconnected) as its health check |
| MongoDB Atlas | The production database |
| Cloudinary | Image uploads; without its keys, uploads go to `uploads/` on local disk |
| Brevo | Password reset emails |

## Required concepts, where each one lives

**Node/Express REST API.** [src/app.js](src/app.js) mounts one router per feature under [src/features/](src/features/); each route lists its access guards inline. Every error leaves through [errorHandler](src/middleware/error.js#L43) in one shape: `{ error: { code, message, fields? } }`.

**MongoDB with Mongoose.** The models in [src/models/](src/models/): User (with a wishlist, a cart and its signed-in sessions), Product, Order (with a status history, discount and refund fields), Review, Coupon, Notification, ReturnRequest and ActivityLog, plus an embedded address schema.

**Register and log in.** Passwords are hashed with bcrypt in a pre-save hook. The session is a JWT in an httpOnly cookie that holds only the user id and a session id; the role is re-read from the database on every request ([auth.js](src/middleware/auth.js)), so an approval or deactivation applies immediately. Each device's session is listed on the account, so signing out ends it on the server too, and a password change signs out every other device.

**CRUD.**

| | Customer | Seller | Rider | Admin |
|---|---|---|---|---|
| Create | Order, review, seller application, wishlist item | Product | | Discount code |
| Read | Catalog, own orders | Own orders, sales stats | Assigned deliveries | All orders, users, sellers, platform stats |
| Update | Profile, password, own review, notification read state | Own product, order status up to shipped | Out for delivery, delivered | Approve or suspend seller, unlist or feature product, turn a discount code on or off, deactivate user |
| Delete | Cancel own order, delete own review, remove wishlist item | Delete or archive own product | | Delete user, remove any review |

**Roles and ownership.** Four roles: customer, seller, rider, admin, plus one owner: an admin who makes and removes admins and picks which areas (users, sellers, products, orders, discount codes) each one manages. [requireRole](src/middleware/auth.js#L78), [requireArea](src/middleware/auth.js#L89) and [requireApprovedSeller](src/middleware/auth.js#L99) guard routes, and every controller that loads a record by id filters by owner, returning 404 rather than 403 so ids can't be probed. Write controllers copy fields through [pick](src/lib/request.js#L4), so a request can't set `role`, `seller` or `ratingAvg`.

**Validation.** The Mongoose validators are the final check behind the client's Zod schemas, and field errors come back in `fields` so the client can show them on the right input.

## Notable decisions

**Discount codes are priced twice, both times on the server.** The checkout preview and the order itself share [priceCoupon](src/features/coupons/service.js), so they can't disagree. The use is claimed with a conditional update inside the checkout transaction, so the last use of a limited code can't be taken twice and a rejected code rolls back the stock reservation. The discount is split across the per-shop orders so each shop's order shows its real share.

**One checkout, one order per shop.** Each shop ships its own items, so [createOrders](src/features/orders/controller.js#L30) splits the cart by seller inside a MongoDB transaction. The client sends only product ids and quantities; prices come from the database. Stock is reserved with a conditional `$inc` (`stock >= qty`), so two buyers can't both get the last unit, and one failed item rolls back the whole checkout. Transactions need a replica set, which is why local development runs an in-memory replica set rather than a standalone `mongod`.

**Only the rider marks an order delivered.** The shop's part ends at shipped, which hands the order to the best-rated rider who is free, before any rider gets a second delivery. The rider then marks it out for delivery and delivered ([deliveries](src/features/deliveries/controller.js)). Neither the shop nor the customer can set delivered, since that starts the return window. If a rider never updates an order, it completes itself 7 days after shipping.

**Status changes can't race.** [cancelOrder](src/features/orders/controller.js#L184) and [advanceOrder](src/features/orders/controller.js#L242) update with a status condition in the filter, so a cancel and a "mark shipped" arriving together can't both succeed, and a cancel can't restock twice.

**Every admin action is logged and undoable.** Each one writes an [ActivityLog](src/models/ActivityLog.js) entry in the same transaction. The owner can [undo](src/features/activity/service.js#L198) it, but only if nothing has changed since; an admin's order cancel is logged but final, because the stock may already have sold again.

**Archive instead of delete.** A product that appears in any order is unlisted rather than deleted, because orders keep a snapshot of the name and price but still reference it. A product an admin unlisted can only be relisted by an admin.

## Tests

```bash
npm test
```

The suite ([test/](test/)) runs against a seeded in-memory replica set and covers the rules that would be expensive to break:
- registration ignoring a submitted `role`; login giving the same answer for a wrong password and an unknown email
- checkout pricing, splitting and stock reservation, and two shoppers racing for the last unit
- a checkout still reporting success when saving the address afterwards fails, so a retry can't buy twice
- an out-of-stock item rolling back the whole checkout
- order privacy between customers and shops
- forward-only status changes
- cancel rules and restocking
- verified-buyer reviews and rating updates
- seller ownership of products
- rejection of foreign asset URLs, and uploads checked by their bytes, so a renamed HTML file can't pass as an image
- suspension unlisting a seller's products
- admins unable to change the owner or hand out admin; the owner making, deactivating and removing admins
- the activity log: every admin action recorded, undo for the owner only (including restoring a deleted account), and a refused undo once the thing has changed again
- sessions ending on sign-out, password change or reset, and deactivation; one-time, expiring reset links that two simultaneous requests can't both use; the 72-byte password limit bcrypt actually reads; the password check on email changes; closing an account with open orders or returns
- returns: the 7-day window, allowed reasons, quantities across partial returns, a double-submitted request, refunds never above what was paid, escalation, and restocking once
- suspending a shop cancelling its unshipped orders, the undo notices, rider hand-over and delivery, and the 7-day auto-complete
- discount codes: validation, one use per customer, a race for the last use, the preview refusing carts checkout would refuse, and the use coming back when a checkout is cancelled
- seller applications and account deletion rules
- the cart: stored on the account, every checkout rule applied when adding, and checkout removing only what was bought
- search matching words across names, categories and specs, and unknown sort names
- the health check failing when the database is down

[CI](.github/workflows/ci.yml) runs lint and the tests on every push and pull request to `main`.

```bash
npm run lint
```

## Known limitations

- Payment is simulated; nothing is charged.
- Without Cloudinary keys, uploads go to `uploads/` on the API's disk, which does not persist on hosts like Render's free tier.
- The only email is the password reset link, sent through Brevo. Order and seller updates arrive as in-app notifications.
- Render's free tier sleeps when idle, so the first request after a quiet spell takes 30-60 seconds.
- Stats bucket days in `Asia/Manila`, and the currency is fixed to PHP.
