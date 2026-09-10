```
profiles
────────────────────────────────
id               uuid
clerk_user_id    text       UNIQUE
first_name       text
last_name        text
email            text
avatar_url       text
is_admin         boolean    DEFAULT false
created_at       timestamptz DEFAULT now()
```

```
create table public.profiles (
  id uuid primary key default gen_random_uuid(),

  clerk_user_id text not null unique,

  first_name text,
  last_name text,
  email text,

  avatar_url text,

  is_admin boolean not null default false,

  created_at timestamptz not null default now()
);

alter table public.profiles enable row level security;
```
