# OSTEN RESTO — Offline-First POS (Expo + SQLite)

A tablet-first point-of-sale app built with React Native and Expo, adapted from a
Figma design ("Food POS Dark — Tablet Device"). Dishes, cart, and order history
all live in an on-device SQLite database, so the app works fully offline and
preserves every record across app restarts.

## Tech

- Expo SDK 57 · React Native 0.86 · React 19
- Expo Router (file-based routing under `src/app`)
- `expo-sqlite` for the embedded relational database
- TypeScript

## Running it

```bash
npm install
npx expo start
```

Scan the QR code with Expo Go on a physical device. No development build is
needed because `expo-sqlite` ships inside Expo Go.

On Windows PowerShell, call `npm.cmd` and `npx.cmd` instead of `npm` and `npx`
if script execution is blocked by policy.

## Database

The database file is `pos_inventory.db`. All access goes through
`src/services/db.ts`, which owns the schema, seeding, and every query.

| Table | Purpose |
| --- | --- |
| `categories` | Menu categories, ordered |
| `dishes` | Menu items: name, category, price, stock, emoji |
| `cart_items` | The open order, one row per dish |
| `orders` | Completed orders with totals |
| `order_items` | Line items for each completed order |
| `app_state` | Key/value session state (order type, discount) |

Schema notes:

- `PRAGMA journal_mode = WAL` for read/write performance.
- `idx_dishes_name` indexes the column the search query filters on.
- Seeding runs only when `categories` and `dishes` are empty, so a reinstall
  resets but a restart does not.
- Deleting a dish cascades to its cart row via `ON DELETE CASCADE`.
- Placing an order writes the order and its lines inside
  `withExclusiveTransactionAsync`, then clears the cart.

Every query that touches user input uses bound parameters. Search is
`WHERE name LIKE ?` with the pattern passed as a parameter, never concatenated
into the SQL string.

## Structure

```
src/
  app/
    _layout.tsx          SQLiteProvider + schema init + CartProvider
    (tabs)/index.tsx     POS screen: search, categories, dish grid
    (tabs)/orders.tsx    Order history from SQLite
    (tabs)/reports.tsx   Revenue and best sellers
    (tabs)/settings.tsx  Restaurant and database info
    payment.tsx          Payment method, discount, place order
  components/            DishCard, AddDishModal, OrderSheet, CategoryTabs
  hooks/
    useDishes.ts         SQL-backed dish state, search, CRUD
    useCart.tsx          SQL-backed cart and orders
  services/db.ts         Schema, seed data, and all queries
  theme/index.ts         Colors, spacing, typography
  lib/format.ts          Money and date formatting
```

## CRUD

- **Create** — "Add Item" in the POS header opens a form that inserts a dish.
- **Read** — the grid renders rows from `SELECT * FROM dishes`, filtered by
  category or search.
- **Update** — the stepper on each card runs `UPDATE dishes SET stock = MAX(0, stock + ?)`;
  quantity and notes in the order sheet update `cart_items`.
- **Delete** — the trash button confirms, then deletes the row.

## Offline durability

The database is a file on the device, so no network request is involved in
reading or writing. To verify: add dishes, enable Airplane Mode, fully kill
Expo Go, relaunch, and the records are still there.

## Design reference

`design/figma-cover.webp` is the source design's cover art. Colors are
approximated from that image and centralized in `src/theme/index.ts`.