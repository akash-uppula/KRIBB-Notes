# Kribb — Clerk Authentication Notes

## 1. What is Clerk?

Clerk is the authentication and user-management service we use in Kribb.

Instead of building authentication ourselves, Clerk handles things such as:

- User registration
- Sign in
- Sign out
- Sessions
- Email verification
- User information
- Authentication state
- Secure token/session persistence

Our React Native / Expo app communicates with Clerk through the `@clerk/expo` package.

---

# 2. Packages We Installed

We installed:

```bash
npx expo install @clerk/expo expo-secure-store
```

### `@clerk/expo`

Provides Clerk functionality for Expo/React Native.

It gives us hooks and APIs such as:

```tsx
useAuth()
useSignIn()
useSignUp()
useUser()
```

### `expo-secure-store`

Used to securely persist Clerk's authentication token/session information on the device.

---

# 3. Clerk Publishable Key

We created a Clerk application and obtained the Publishable Key.

In `.env`:

```env
EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_your_key_here
```

Important:

- The Publishable Key can be used in the client application.
- Never put the Clerk Secret Key inside the React Native application.
- `EXPO_PUBLIC_` allows Expo to expose the variable to the app.

---

# 4. ClerkProvider

Our root layout contains:

```tsx
import { ClerkProvider } from "@clerk/expo";
import { tokenCache } from "@clerk/expo/token-cache";
import { Slot } from "expo-router";

const publishableKey =
  process.env.EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY!;

if (!publishableKey) {
  throw new Error("Add your Clerk Publishable Key to the .env file");
}

export default function RootLayout() {
  return (
    <ClerkProvider
      publishableKey={publishableKey}
      tokenCache={tokenCache}
    >
      <Slot />
    </ClerkProvider>
  );
}
```

## What does ClerkProvider do?

`ClerkProvider` makes Clerk's authentication state and APIs available to the rest of the application.

Conceptually:

```text
ClerkProvider
      |
      v
 Expo Router
      |
      +-- Auth screens
      |
      +-- Protected Kribb screens
           |
           +-- Home
           +-- Search
           +-- Saved
           +-- Profile
```

Because the provider is at the top of the application, child screens can use Clerk hooks.

---

# 5. tokenCache

We use:

```tsx
import { tokenCache } from "@clerk/expo/token-cache";
```

and pass it to:

```tsx
<ClerkProvider
  publishableKey={publishableKey}
  tokenCache={tokenCache}
>
```

The cache allows Clerk to persist authentication information between app launches.

Conceptually:

```text
User signs in
     |
     v
Clerk creates session
     |
     v
tokenCache persists session information
     |
     v
App closes
     |
     v
App opens again
     |
     v
Clerk restores authentication state
```

This is why users can remain signed in instead of having to log in every time the app opens.

---

# 6. Our Kribb Route Structure

We organized the application like this:

```text
src/app
|
+-- _layout.tsx
|
+-- index.tsx
|
+-- (auth)
|   |
|   +-- _layout.tsx
|   +-- sign-in.tsx
|   +-- sign-up.tsx
|
+-- (root)
    |
    +-- _layout.tsx
    |
    +-- (tabs)
        |
        +-- _layout.tsx
        +-- index.tsx       -> Home
        +-- search.tsx
        +-- saved.tsx
        +-- profile.tsx
```

The parentheses mean Expo Router route groups.

They help organize routes without becoming part of the actual URL/path.

---

# 7. Authentication Flow

Our overall authentication flow is:

```text
                    Kribb starts
                         |
                         v
                    app/index.tsx
                         |
                         v
                     useAuth()
                         |
                 Is Clerk loaded?
                    /         \
                  NO           YES
                  |             |
                wait       Is signed in?
                            /       \
                          YES        NO
                           |          |
                           v          v
                         Tabs      Sign In
                                      |
                                +-----+-----+
                                |           |
                              Sign In     Sign Up
                                |           |
                                |       Verification
                                |           |
                                +-----+-----+
                                      |
                                      v
                                  Signed in
                                      |
                                      v
                                     Home
```

---

# 8. `useAuth()`

Import:

```tsx
import { useAuth } from "@clerk/expo";
```

Then:

```tsx
const { isLoaded, isSignedIn } = useAuth();
```

`useAuth()` is mainly used to work with authentication/session state.

## `isLoaded`

Tells us whether Clerk has finished loading the authentication state.

```tsx
if (!isLoaded) {
  return null;
}
```

We wait before making authentication decisions.

Why?

When the app starts, Clerk needs a moment to determine whether an existing session exists.

```text
isLoaded = false
       |
       v
Clerk is still checking

isLoaded = true
       |
       v
Authentication state is ready
```

## `isSignedIn`

Tells us whether the user currently has an authenticated session.

```text
isSignedIn = true
      |
      v
User is authenticated

isSignedIn = false
      |
      v
User is not authenticated
```

---

# 9. Root `index.tsx`

Our root entry point checks authentication:

```tsx
import { useAuth } from "@clerk/expo";
import { Redirect } from "expo-router";

const Index = () => {
  const { isLoaded, isSignedIn } = useAuth();

  if (!isLoaded) {
    return null;
  }

  if (isSignedIn) {
    return <Redirect href="/(root)/(tabs)" />;
  }

  return <Redirect href="/(auth)/sign-in" />;
};

export default Index;
```

The job of this screen is not to display UI.

It decides where the user should go.

```text
App starts
   |
   v
index.tsx
   |
   +-- signed in ------> Tabs
   |
   +-- signed out -----> Sign In
```

---

# 10. Protecting the `(root)` Area

We created:

```text
src/app/(root)/_layout.tsx
```

with:

```tsx
import { useAuth } from "@clerk/expo";
import { Redirect, Slot } from "expo-router";

const RootLayout = () => {
  const { isLoaded, isSignedIn } = useAuth();

  if (!isLoaded) {
    return null;
  }

  if (!isSignedIn) {
    return <Redirect href="/(auth)/sign-in" />;
  }

  return <Slot />;
};

export default RootLayout;
```

This is our protected route.

Why do we need this if `index.tsx` already checks authentication?

Because users could potentially navigate directly to a protected route.

So `(root)/_layout.tsx` acts as a second security/navigation guard.

```text
(root)
   |
   v
Authentication check
   |
   +-- signed in ------> render protected screens
   |
   +-- signed out -----> Sign In
```

---

# 11. `useSignIn()`

Import:

```tsx
import { useSignIn } from "@clerk/expo";
```

Then:

```tsx
const { signIn } = useSignIn();
```

`useSignIn()` gives us the object used to perform the sign-in flow.

It is different from `useAuth()`.

### `useAuth()`

Asks:

> What is the current authentication state?

```tsx
const { isLoaded, isSignedIn } = useAuth();
```

### `useSignIn()`

Performs:

> I want to authenticate a user.

```tsx
const { signIn } = useSignIn();
```

---

# 12. Current Sign In Flow

Our current Sign In flow uses the current Clerk API:

```tsx
const { createError } = await signIn.create({
  identifier: email,
});
```

Then:

```tsx
const { error: passwordError } = await signIn.password({
  password,
});
```

Then we check:

```tsx
if (signIn.status === "complete") {
  ...
}
```

Finally:

```tsx
await signIn.finalize({
  navigate: () => router.replace("/(root)/(tabs)"),
});
```

The complete flow:

```text
Email
  +
Password
  |
  v
signIn.create()
  |
  v
signIn.password()
  |
  v
Check signIn.status
  |
  +-- complete
  |
  v
signIn.finalize()
  |
  v
Create/activate authenticated session
  |
  v
router.replace()
  |
  v
Home
```

---

# 13. Why `signIn.create()`?

```tsx
await signIn.create({
  identifier: email,
});
```

This starts the sign-in process using the supplied identifier.

In our case:

```tsx
identifier: email
```

The identifier is the user's email address.

---

# 14. Why `signIn.password()`?

```tsx
await signIn.password({
  password,
});
```

This supplies the password for the sign-in attempt.

We separated the sign-in process into the current Clerk API steps rather than using the old legacy pattern.

---

# 15. `signIn.status`

After the authentication steps:

```tsx
if (signIn.status === "complete") {
```

`complete` means Clerk has reached a state where the sign-in can be finalized.

There can be other authentication states in more complicated flows, such as when additional verification is required.

For our current email/password flow, we expect:

```text
signIn.status
      |
      v
"complete"
      |
      v
finalize()
```

---

# 16. `signIn.finalize()`

We use:

```tsx
await signIn.finalize({
  navigate: () => router.replace("/(root)/(tabs)"),
});
```

This completes the sign-in flow and activates the session.

The `navigate` callback tells Clerk what to do after successful authentication.

Our callback:

```tsx
navigate: () => router.replace("/(root)/(tabs)")
```

takes the user to the default tab, which is our Home screen.

---

# 17. `useRouter()`

We use:

```tsx
import { useRouter } from "expo-router";

const router = useRouter();
```

Then:

```tsx
router.replace("/(root)/(tabs)");
```

`replace()` replaces the current route instead of adding another route to the navigation history.

This is useful after authentication because we don't want the user to return to the Sign In screen by simply navigating backward.

---

# 18. `Link`

For normal navigation between authentication screens we use:

```tsx
import { Link } from "expo-router";
```

Example:

```tsx
<Link href="/(auth)/sign-up">
  Don't have an account? Sign Up
</Link>
```

And:

```tsx
<Link href="/(auth)/sign-in">
  Already have an account? Sign In
</Link>
```

### `Link` vs `router`

Use `Link` when the user is tapping a visible navigation link.

Use `router` when navigation happens programmatically after some logic.

Example:

```text
User taps Sign Up
       |
       v
<Link />
```

versus:

```text
Authentication succeeds
       |
       v
router.replace()
```

---

# 19. `useSignUp()`

Import:

```tsx
import { useSignUp } from "@clerk/expo";
```

Then:

```tsx
const { signUp } = useSignUp();
```

`useSignUp()` gives us the object used to perform the registration flow.

---

# 20. Current Sign Up Flow

Our current flow is:

```text
Email + Password
      |
      v
signUp.password()
      |
      v
signUp.verifications.sendEmailCode()
      |
      v
Email verification code
      |
      v
signUp.verifications.verifyEmailCode()
      |
      v
signUp.finalize()
      |
      v
Signed in
      |
      v
Home
```

---

# 21. `signUp.password()`

We use:

```tsx
const { error } = await signUp.password({
  emailAddress: email,
  password,
});
```

This starts the password-based sign-up flow.

Notice that sign-up uses:

```tsx
emailAddress
```

while the sign-in identifier is:

```tsx
identifier
```

---

# 22. `signUp.verifications.sendEmailCode()`

After starting registration:

```tsx
const { error: sendError } =
  await signUp.verifications.sendEmailCode();
```

This asks Clerk to send a verification code to the user's email.

Conceptually:

```text
User enters email
       |
       v
Clerk
       |
       v
Send verification code
       |
       v
User's email
```

---

# 23. `signUp.verifications.verifyEmailCode()`

After the user receives the code:

```tsx
const { error } =
  await signUp.verifications.verifyEmailCode({
    code,
  });
```

This checks whether the code entered by the user is valid.

```text
User enters code
      |
      v
verifyEmailCode()
      |
      +-- valid ------> continue
      |
      +-- invalid ----> error
```

---

# 24. `signUp.finalize()`

After successful verification:

```tsx
const { error: finalizeError } = await signUp.finalize({
  navigate: () => router.replace("/(root)/(tabs)"),
});
```

This completes the sign-up process and establishes the authenticated session.

Then the user is sent to Home.

---

# 25. `useUser()`

On the Profile screen we use:

```tsx
import { useUser } from "@clerk/expo";

const { user } = useUser();
```

`useUser()` gives us information about the currently authenticated user.

For example:

```tsx
user?.firstName
```

and:

```tsx
user?.primaryEmailAddress?.emailAddress
```

We use optional chaining (`?.`) because user data may not be available at the exact moment the component first renders.

---

# 26. `signOut()`

On Profile we use:

```tsx
const { signOut } = useAuth();
```

Then:

```tsx
await signOut();
```

This ends the current authenticated session.

Our flow:

```text
Profile
   |
   v
Sign Out
   |
   v
signOut()
   |
   v
Session ends
   |
   v
isSignedIn = false
   |
   v
Protected root detects signed-out state
   |
   v
Sign In
```

We can also explicitly navigate:

```tsx
router.replace("/(auth)/sign-in");
```

after sign-out.

---

# 27. Complete Kribb Authentication Cycle

```text
                         KRIBB
                           |
                           v
                      app/index.tsx
                           |
                     useAuth()
                           |
                 +---------+---------+
                 |                   |
             Signed Out          Signed In
                 |                   |
                 v                   v
              Sign In              Tabs
                 |                   |
          +------+-------+            +------------------+
          |              |            |        |        |
       Sign In        Sign Up       Home    Search    Saved
          |              |                              |
          |              v                              |
          |        signUp.password()                    |
          |              |                              |
          |        sendEmailCode()                      |
          |              |                              |
          |        Verify Code                          |
          |              |                              |
          |        verifyEmailCode()                    |
          |              |                              |
          |        finalize()                           |
          |              |                              |
          +--------------+------------------------------+
                         |
                         v
                     Signed In
                         |
                         v
                        Home
                         |
                         v
                      Profile
                         |
                         v
                      signOut()
                         |
                         v
                    Signed Out
                         |
                         v
                      Sign In
```

---

# 28. Important Difference Between the Hooks

| Hook | Main Purpose |
|---|---|
| `useAuth()` | Authentication state and session actions |
| `useSignIn()` | Sign-in process |
| `useSignUp()` | Account creation process |
| `useUser()` | Current user's information |

Think about them like this:

```text
useAuth()
   -> "Am I logged in?"
   -> "Is Clerk loaded?"
   -> "Sign me out."

useSignIn()
   -> "Log this user in."

useSignUp()
   -> "Create this user's account."

useUser()
   -> "Give me information about the logged-in user."
```

---

# 29. Authentication vs Authorization

These are different concepts.

### Authentication

> Who are you?

Clerk handles this for us.

```text
Email
Password
Verification
Session
```

### Authorization

> What are you allowed to do?

This will become important later in Kribb.

For example:

```text
User
  |
  +-- Can view their flat
  +-- Can view rooms
  +-- Can add expenses

Admin
  |
  +-- Can manage all rooms
  +-- Can manage members
  +-- Can remove users
```

We'll deal with authorization when Kribb's actual data model is built.

---

# 30. Error Handling

We consistently use:

```tsx
const { error } = await ...
```

Then:

```tsx
if (error) {
  Alert.alert("Something went wrong", error.message);
  return;
}
```

This is preferable to blindly continuing after an authentication operation fails.

For example:

```tsx
const { error } = await signUp.password({
  emailAddress: email,
  password,
});

if (error) {
  Alert.alert("Sign Up Failed", error.message);
  return;
}
```

The `return` is important because it stops the rest of the function from running.

---

# 31. Why We Don't Use `any`

Because Kribb is TypeScript:

```text
.tsx -> React components/screens
.ts  -> utilities/types/hooks/config
```

We want Clerk's TypeScript types to work for us.

We should avoid:

```tsx
const error: any = ...
```

and avoid disabling TypeScript/ESLint just to make an error disappear.

If VS Code highlights a Clerk method, we should check the current Clerk API rather than forcing TypeScript to accept it.

---

# 32. Legacy APIs We Encountered

During development we initially encountered older Clerk examples such as:

```tsx
signIn.create({
  identifier: email,
  password,
});
```

and:

```tsx
signUp.prepareVerification()
```

and:

```tsx
signUp.verifyEmailCode()
```

For our current Clerk Expo SDK, we moved to the current API structure:

```tsx
signIn.create()
signIn.password()
signIn.finalize()
```

and:

```tsx
signUp.password()
signUp.verifications.sendEmailCode()
signUp.verifications.verifyEmailCode()
signUp.finalize()
```

This is an important lesson:

> Always check the API version installed in the project and the current documentation when a TypeScript/lint error appears.

---

# 33. Our Current Clerk Mental Model

The easiest way to remember everything:

```text
                  ClerkProvider
                       |
              Gives app access to Clerk
                       |
        +--------------+--------------+
        |              |              |
     useAuth()    useSignIn()    useSignUp()
        |              |              |
   Auth state       Sign In        Sign Up
        |
        +---- signOut()
        |
        +---- isLoaded
        |
        +---- isSignedIn

              useUser()
                   |
                   v
             Current user
```

---

# 34. Final Kribb Flow

At this stage, Kribb has:

```text
              START APP
                  |
                  v
              ClerkProvider
                  |
                  v
              useAuth()
                  |
          +-------+-------+
          |               |
       Signed out      Signed in
          |               |
          v               v
       Sign In          Home
          |               |
          |             Tabs
          |               |
       Sign Up       +----+----+----+
          |          |    |    |    |
          |        Home Search Saved Profile
          |
    Email Verification
          |
          v
       finalize()
          |
          v
       Signed In
          |
          v
         Home
          |
          v
       Profile
          |
          v
       signOut()
          |
          v
       Sign In
```

This is the authentication foundation we will build the rest of Kribb on.
