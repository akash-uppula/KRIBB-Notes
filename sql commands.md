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

```
create policy "Users can view their own profile"
on public.profiles
for select
to authenticated
using (
  clerk_user_id = auth.jwt()->>'sub'
);
```

```
create policy "Users can update their own profile"
on public.profiles
for update
to authenticated
using (
  clerk_user_id = auth.jwt()->>'sub'
)
with check (
  clerk_user_id = auth.jwt()->>'sub'
);
```

```
create policy "Users can create their own profile"
on public.profiles
for insert
to authenticated
with check (
  clerk_user_id = auth.jwt()->>'sub'
  and is_admin = false
);
```

```
create or replace function public.prevent_client_admin_change()
returns trigger
language plpgsql
as $$
begin
  if new.is_admin is distinct from old.is_admin
     and auth.role() <> 'service_role' then
    raise exception 'is_admin cannot be changed by the client';
  end if;

  return new;
end;
$$;
```

```
create trigger protect_is_admin
before update on public.profiles
for each row
execute function public.prevent_client_admin_change();
```

```
create table public.properties (
id uuid primary key default gen_random_uuid(),

clerk_user_id text not null,

title text not null,
description text,

price numeric not null,

type text not null
check (type in ('apartment', 'house', 'villa', 'studio')),

bedrooms int not null default 1,
bathrooms int not null default 1,

area_sqft int,

address text not null,
city text not null,

latitude double precision,
longitude double precision,

images text[] not null default '{}',

is_featured boolean not null default false,
is_sold boolean not null default false,

created_at timestamptz not null default now()
);

```

```
alter table public.properties enable row level security;
```

```
create policy "Authenticated users can view properties"
on public.properties
for select
to authenticated
using (true);
```

```
create policy "Users can create their own properties"
on public.properties
for insert
to authenticated
with check (
  clerk_user_id = (select auth.jwt()->>'sub')
);
```

```
create policy "Users can update their own properties"
on public.properties
for update
to authenticated
using (
  clerk_user_id = (select auth.jwt()->>'sub')
)
with check (
  clerk_user_id = (select auth.jwt()->>'sub')
);
```

```
create policy "Users can delete their own properties"
on public.properties
for delete
to authenticated
using (
  clerk_user_id = (select auth.jwt()->>'sub')
);
```

```
create table public.saved_properties (
  id uuid primary key default gen_random_uuid(),

  clerk_user_id text not null,

  property_id uuid not null
    references public.properties(id)
    on delete cascade,

  created_at timestamptz not null default now(),

  unique (clerk_user_id, property_id)
);


alter table public.saved_properties enable row level security;


create policy "Users can view their own saved properties"
on public.saved_properties
for select
to authenticated
using (
  clerk_user_id = (select auth.jwt()->>'sub')
);


create policy "Users can save properties"
on public.saved_properties
for insert
to authenticated
with check (
  clerk_user_id = (select auth.jwt()->>'sub')
);


create policy "Users can remove their saved properties"
on public.saved_properties
for delete
to authenticated
using (
  clerk_user_id = (select auth.jwt()->>'sub')
);

```

```
create policy "Users can upload property images"
on storage.objects
for insert
to authenticated
with check (
  bucket_id = 'property-images'
  and (storage.foldername(name))[1] = (select auth.jwt()->>'sub')
);
```

```
create policy "Users can view property images"
on storage.objects
for select
to authenticated
using (
  bucket_id = 'property-images'
);
```

```
create policy "Users can update their property images"
on storage.objects
for update
to authenticated
using (
  bucket_id = 'property-images'
  and (storage.foldername(name))[1] = (select auth.jwt()->>'sub')
)
with check (
  bucket_id = 'property-images'
  and (storage.foldername(name))[1] = (select auth.jwt()->>'sub')
);
```

```
create policy "Users can delete their property images"
on storage.objects
for delete
to authenticated
using (
  bucket_id = 'property-images'
  and (storage.foldername(name))[1] = (select auth.jwt()->>'sub')
);
```
