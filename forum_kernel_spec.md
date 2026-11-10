# Forum Kernel Panic

## Workflow

One step of spec at a time:

1. I write the step and I read it again.
2. The rest is me, just this step, nothing more.
3. I run the tests / checks by hand and a save the output.
4. I ask claude review of diff (not implementation)
5. commit, check the step.

### Rules for Claude (agent)

- Reads the spec and replies only for the step I name.
- Doesn't write code in files of the project. Just questions, hints, pseudocode and review.
- If I stack for > 2 rounds: smaller example of another problem, never the solution.
- Doesn't respond "it passes". I run the tests and I paste the output.
- If the spec is not in line with the code: claude points it, and code (reality) wins until i fix it.

## 0. Prerequisites
| Tool        | Why                          |
| ----------- | ---------------------------- |
| Go 1.27+    | lang                         |
| gcc         | `mattn/go-sqlite3` needs cgo |
| sqlite3 CLI | to look the database by hand |
| Docker      | project submit               | 

- Allowed packages: std lib, `mattn/go-sqlite3`, `x/crypto/bcrypt`, `gofrs/uuid` or `google/uuid`.
- Prohibited: frontend frameworks (React, Vue, ...)
- The repo is: `https://platform.zone01.gr/git/mzervini/forum.git` `.db` goes into `.gitignore`
- add gorilla/websocket, marked "bonus branch only", since it's outside the allowed list.

## 1. Foundation

### 1.1 Layout
```text
forum/
├── cmd/
│   └── forum/
│       └── main.go          [1] wiring only: open DB, build router, listen. No logic.
├── internal/
│   ├── db/                  opens SQLite and applies the schema. Knows no tables' meaning.
│   │   ├── db.go            [1] Open(), PRAGMA foreign_keys = ON
│   │   └── migrations/
│   │       ├── 001_init.sql             [1] the 8 tables
│   │       └── 002_seed_categories.sql  [1] the 9 launch categories
│   ├── store/               the ONLY place with SQL. Returns structs, knows no HTTP.
│   │   ├── users.go         [2]
│   │   ├── sessions.go      [2]
│   │   ├── posts.go         [3]
│   │   ├── categories.go    [3]
│   │   ├── comments.go      [4]
│   │   ├── likes.go         [5]
│   │   └── *_test.go        written together with each file
│   ├── middleware/          wraps handlers. Identifies, never decides.
│   │   ├── recover.go       [1] panic -> 500
│   │   ├── logging.go       [1]
│   │   ├── session.go       [2] cookie -> user (or guest) in the request
│   │   ├── require.go       [2] RequireLogin -> 401
│   │   ├── csrf.go          [2]
│   │   ├── limits.go        [7] body size -> 413
│   │   └── ratelimit.go     [7] -> 429
│   └── handlers/            HTTP only: parse, validate, call store, render.
│       ├── routes.go        [1] the route table
│       ├── render.go        [1] template helper
│       ├── errors.go        [1] error pages (400, 401, 403, 404, 405, 413, 429, 500)
│       ├── auth.go          [2] register, login, logout
│       ├── posts.go         [3] list, view, create (+ filters at [6])
│       ├── comments.go      [4]
│       └── likes.go         [5]
├── templates/
│   ├── base.html            [1]
│   ├── home.html            [1]
│   ├── error.html           [1]
│   ├── register.html, login.html     [2]
│   └── post.html, new_post.html      [3]
├── static/
│   └── css/style.css        [1]
├── docs/
│   └── schema.dbml          [0]
├── data/                    runtime only, gitignored: forum.db lives here
├── Dockerfile               [1]
├── .dockerignore, .gitignore  [1] (*.db and data/ ignored)
├── go.mod, go.sum
└── README.md                [8]
```
**Imports**
```text
cmd/forum  ->  db, store, middleware, handlers
handlers   ->  store, middleware
middleware ->  store
store      ->  (std lib only)
db         ->  (std lib + sqlite driver)
```

- Why `cmd/forum/main.go` only wires: logic outside `main` can be tested without starting a server.
- Why `internal/`: the compiler forbids other modules from importing it, so nothing outside depends on my internals.
- Why SQL only in `store`: a schema or query change touches one package, and handlers never see SQL.
- Why one-way imports: no import cycles, and `store` has no HTTP, so its tests are plain Go.
- Check: all folders and empty files exist, `go build ./...` passes, the server starts and answers `/`.
- Decision: CSRF middleware and the hidden token in templates come with the first POST form (phase 2). Rate limit and size limits stay in phase 7.
- Decisions: templates are read from disk (no `embed`); `.db` lives in `data/`; no `models` package (structs live in `store`); schema and seed are numbered `.sql` files.
- It says migration 001_init.sql creates "the 8 tables", but 1.2 now lists 10. Keep 001 as the 8, and add 003_notifications.sql and 004_messages.sql on their feature branches.
- The tree has no files for notifications or chat. Add store/notifications.go, handlers/notifications.go, store/messages.go, and handlers/chat.go with a small hub, each tagged with its phase.
  
### 1.2 Database

- Tables: `users`, `sessions`, `posts`, `comments`, `post_likes`, `comment_likes`, `categories`, `post_categories`, `notifications`, `messages`
- Scheme: `docs/schema.dbml`
- Everything 'NOT NULL' except if I say so. `value` in likes with CHECK - `PRAGMA foreign_keys = ON`.
- Why split tables for `post_likes` & `comment_likes`:
  - Real foreign keys: `post_likes.post_id` points only to `posts`, and `comment_likes.comment_id` points only to `comments`. With **ON DELETE CASCADE**, deleting a post also removes its likes.
- The alternatives are worse:
  - One likes table with both `post_id` and `comment_id`, where one is always **NULL**, needs a CHECK for "exactly one of the two". NULLs in a unique/PK constraint also don't conflict with each other (SQLite treats them as distinct values), so uniqueness breaks.
  - One table with `target_type` and `target_id` (polymorphic) can't have a FK, because `target_id` points to two different tables. Integrity then depends on your code.
- Simpler queries: `COUNT(*) ... WHERE post_id = ?` needs no type filter.
- Schema: `users.role` with `CHECK` in (`user`, `mod`, `admin`), and `users.banned_at` (NULL = not banned).
- v2 tables: `post_reports` and `comment_reports` (same reasoning as the likes tables: real foreign keys, no polymorphic target).

Why a composite PK (`user_id`, `post_id`):
- One like per user per post, enforced by the database: the pair is unique by definition. If a user double-clicks or sends two parallel requests, the second `INSERT` fails. A check in Go (`SELECT`, then `INSERT`) has a race condition.
- No surrogate `id` needed: no other table references a like row. An extra `id` would be a useless column, and you'd still need `UNIQUE(user_id, post_id)` to get the same guarantee.
- Free index: the PK indexes "did user X like post Y?". For counting per post (`WHERE post_id = ?`), the leading column is `user_id`, so that query needs its own index on `post_id`.
- Like → dislike: you can do `INSERT ... ON CONFLICT(user_id, post_id) DO UPDATE SET value = ?`, with no duplicate rows.

### 1.3 Server & errors

- Server start with no `panic` and responds to `/`
- Every error returns right HTTP status and error page (400, 401, 403, 404, 405, 413, 429, 500).

## 2. Authentication

### 2.1 Register

- Requests: `email`, `username`, `password`.
- Email already exists -> error response. Known trade-off: this reveals that the email is registered. Accepted.
- `username`: **3-20 characters**
- `password`: **8-72 bytes** (bcrypt cuts at 72 bytes)
- `email`: **valid**, **unique** and **case-insensitive**
- All length limits count characters (runes), except `password` (8-72 **bytes**, because of bcrypt).

### 2.2 Login & sessions

- Wrong email or password ? error response. Same message and status for wrong `email` and wrong `password`
- If the email does not exist, still run a **bcrypt** compare against a fixed fake hash, so both failures take about the same time.
- After **5** failed tries in **15 minutes** per IP and per email, return **429**.
- Count failed tries by the submitted email, whether it exists or not (otherwise 429 reveals which emails exist). Known trade-off: an attacker can block a victim's login for 15 minutes. Accepted.
- One open session per user (new login replaces the old one).
- Cookie with expiration. **Duration: 24h** with the same expiry in the DB
  - Short enough to limit damage: if the cookie is stolen, it stops working after **a day** (banks uses minutes, and "remember me" sites uses **30d**)
  - Long enough to not annoy users
  - The server decides, not the cookie: the real expiry is `expires_at` in `sessions` table. The cookie's `Expires/Max-age` should match it. Check `expires_at` on every request.
  - Fixed vs sliding: fixed means **24h** from login. Sliding means the expiry extends on each activity. Fixed is ok for this project.
  - Session cookie vs persistent: a cookie with no expire dies when the browser closes.

#### Cookie flags 

- `HttpOnly`: Javascript can't read the cookie which limits **XSS** theft.
- `SameSite=Lax`: basic **CSRF** protection.
- `Secure`: only over **HTTPS** (skip it in localhost)
- `Path=/`
- On logout and login: delete the session row and expire the cookie (`MaxAge=-1`). On new login, delete the user's old sessions first.
- Expired session = **guest**.

### 2.3 Roles & permissions

| Action                               | Guest | User | Mod           | Admin          |
| ------------------------------------ | ----- | ---- | ------------- | -------------- |
| Reads `posts`/`comments`/`counts`    | +     | +    | +             | +              |
| Register / Login                     | +     | n/a  | n/a           | n/a            |
| Creates `post` / `comment`           | x     | +    | +             | +              |
| Like / dislike                       | x     | +    | +             | +              |
| Filters **mine** / **liked**         | x     | +    | +             | +              |
| **Edit** / **delete** owned          | x     | +    | +             | +              |
| **Delete** others `post` / `comment` | x     | x    | +             | +              |
| Report `post` / `comment`            | x     | +    | +             | +              |
| Read & close reports                 | x     | x    | +             | +              |
| `Ban` / `suspend` user               | x     | x    | + (only User) | + (User + Mod) |
| Change roles, manage categories      | x     | x    | x             | +               |

- How first admin starts: **Not** from register form. Seed in startup from `env` var, or `UPDATE` in DB.
- `mod` can delete content and ban only `User` accounts, never another `mod` or `admin`.
- What banned sees: only read `posts` / `comments`
- Ban removes sessions.
- Scope: v1 = Guest + User only. Mod, Admin, ban, reports and edit/delete are v2. Only `users.role` and `users.banned_at` exist in v1.
- Notifications, sort, and chat aren't labelled v1/v2. Say it explicitly: sort and notifications are post-core, chat is the bonus branch.
- 6.1 sits inside "Filters", so ?sort needs its own row in the checks.

## 3. Posts & Categories

- Only registered users create posts.
- One post has 1 or more categories.
- Posts are visible to **all** (guests too)
- Categories are seeded at startup. Launch set (9): `web`, `pwn`, `re`, `net`, `privacy`, `ctf`, `tools`, `newbies`, `meta`. The others are added later by an admin (v2).
- "1 or more categories" is not enforced by the schema: the handler checks it, and inserts the post and its categories in one transaction. Zero categories -> **400**.
- Categories:
  - **Offensive**: `web`, `pwn`, `re`, `net`, `hw`
  - **Defence & theory**: `privacy`, `blue`, `crypto`, `osint`
  - **Practice**: `ctf`, `tools`, `lab`
  - **Community**: `newbies`, `meta`, `careers`, `linux`

## 4. Comments

- Only registered users can comment.
- Comments are visible to **all**

## 5. Likes & Dislikes

- Only registered users can like/dislike posts and comments.
- Counts are visible to **all**.
- Second like from the same user in the same `item`: **toggle** like.
- Dislike from the same user who liked an `item`: **toggle** like to dislike.
- Same `value` again: **remove**, other `value`: **change**

## 6. Filters

- Per `category` (like subforum) **all**.
- My posts: only logged in.
- Post I liked: only logged in.

### 6.1 Sort
- Sort order for the posts list: `newest`, `top`, `rising`. Applies to everyone (guests too).
- `newest`: most recent first.
- rising: likes received in the last 24h, highest first.
  - Why: it's cheap to compute, and an old post with many likes doesn't stay on top.
  - This needs a created_at column on post_likes. Check that your schema has it.
- top: add a tie-breaker (created_at DESC). Otherwise the order can change between page loads.
- Default: newest.
- Invalid value (?sort=foo): fall back to newest. It's a user-editable URL param, so a 400 is overkill.
- Important: ORDER BY can't take ? params. Map the sort name to a fixed SQL fragment from a whitelist, and never paste the user's string into the query. This ties into your 9.3.

## 7. Docker

- Project builds and runs with `docker build` / `docker run`.
- The volume is the whole `data/` directory (mounted at the same path inside the container), not only `forum.db`. SQLite also writes `-wal` and `-shm` files next to the database, and a single-file mount would lose them.
- Check: create a post, stop and remove the container, run again with the same volume. The post is still there.

## 8. Tests & audit

- Unit tests for:
  - `store/sessions`: an expired session is not valid; a new login replaces the old session.
  - `store/likes`: the same `value` twice removes the row; the other `value` changes it; two inserts for the same pair fail.
  - `store/users`: duplicate email (different case) is rejected.
  - validation: length limits count characters, not bytes (test with a Greek string); password over 72 bytes is rejected.
  - `middleware` (with `httptest`): missing CSRF token -> 403; body over 1 MB -> 413; the 6th failed login in 15 minutes -> 429.
  - `handlers`: a guest `POST` to create a post -> 401.
  - store/notifications: a like creates one notification, an undo deletes it, and no notification for your own post.
  - sort: an unknown value falls back to newest, and the whitelist never reaches the SQL as raw text.
  - chat: an empty message is rejected, a wrong Origin is refused, and a guest gets 401.

## 9. Security

### 9.1 CSRF

- Every `POST` form has a secret token. Missing or wrong -> **403**.
- Logged-in users: the token is bound to the session.
- Guests (login, register): there is no session yet, so the token must match a short-lived cookie set when the form was rendered.
- Check: remove the field in devtools and send. It must fail.

### 9.2 Limits size

- Limit per field: `title` **<= 100**, `post body` **<= 10000**, `comment` **<= 2000 characters**. Above the limit -> **400**.
- Sum limit in body of request: **1 MB**. Above that -> **413**.
- Check: send **1 MB** trash with `curl`.

### 9.3 Escaping / SQL

- Whole user input becomes escaped in templates. All SQL has ? params, no string merging.
- Check: `post` with `<script>alert(1)</script>` and `username ' OR 1=1 --`.

### 9.4 Basics

- Passwords hashed with **bcrypt**
- Session IDs with **UUID**

## 10. Notifications
- Notify on a like, and on a comment on your post. Don't notify on a dislike.
  - Why: negative notifications are noise, and they invite harassment.
- Read state: the badge counts unread, the page lists them, and a POST button "Mark all as read" clears them.
  - Why: a GET that changes state breaks your own CSRF rule (9.1).
- Like undone or changed to a dislike: delete the like notification.
  - Why: otherwise you get a ghost notification, and toggling like/unlike would spam the post's owner.
  - Add a partial unique index on (actor_id, post_id) where type = 'like'.
- Post deleted: ON DELETE CASCADE on post_id removes its notifications. You already do the same for likes.
- Columns: user_id (receiver), actor_id, type with CHECK in (like, comment), post_id, read_at (NULL = unread), created_at.
- Scope: likes on comments are v2.

## 11. Chat (own branch, gorilla/websocket is fine)
- Mechanism: websocket, because you already decided to use it.
  - Why: real-time two-way messages, and no polling load.
- Message length: 1–1000 characters (runes), after trimming whitespace.
- Empty message: reject on the server and don't store it. Over the limit: reject too.
- Recipient offline: store it in the DB first, then push it only if they're connected. They see it when they open the conversation.
- Persist first, then broadcast, so a crash can't lose a message the sender saw as sent.
- Auth: the upgrade request carries the session cookie. Reject it if it's not a valid session (401).
- Origin check: gorilla's CheckOrigin must only allow your own origin. Otherwise another site can open a socket as the logged-in user (cross-site websocket hijacking). Your CSRF middleware doesn't cover websockets.
- Also set a read-size limit on the socket, so one huge frame can't eat memory.
- History: last 50 messages per conversation, plus "load older".
- Columns: sender_id, recipient_id, body, created_at, read_at.
- A user can't message themselves.