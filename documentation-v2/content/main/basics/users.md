---
order: 0
title: "Users"
description: "Learn about users in Lucia"
---

Users are stored in your database and are represented by a `User` object within Lucia. By default, they only hold their user id.

```ts
const user = {
	userId: "laRZ8RgA34YYcgj"
};
```

#### User id

The primary way to identify users is by their user id. It's randomly generated by Lucia and it's 15 characters long. You can of course configure it to [generate your own user id]().

#### User attributes

You can define additional attributes of your users such as email and username.

## Defining user attributes

You can define custom user attributes by returning them in [`getUserAttributes()`]() configuration. As long as the required fields are defined in the user table, you can add any number of fields to the user table.

```ts
import { lucia } from "lucia";

lucia({
	// ...
	getUserAttributes: (databaseUser) => {
		return {
			username: databaseUser.username
		};
	}
});

const user: User = await auth.getUser(userId);
// `userId` is always defined
const { userId, username } = user;

// `getUserAttributes()` params
// `Lucia.databaseUser`: see next section
type DatabaseUser = {
	// user table must have these fields
	id: string;
} & Lucia.databaseUser;
```

You can add additional columns to the user table so long as the `id` column exists as primary key. In the example above, the `username` column is added to the table and is included in `databaseUser`.

### Typing additional fields in the user table

Additional fields in your session table should be defined in [`Lucia.DatabaseUserAttributes`]().

```ts
/// <reference types="lucia" />
declare namespace Lucia {
	// ...
	type DatabaseUserAttributes = {
		// required fields (i.e. id) should not be defined here
		username: string;
	};
}
```

## Create users

You can create new users by calling [`Auth.createUser()`]().

```ts
import { auth } from "./lucia.js";
import { LuciaError } from "lucia";

try {
	await auth.createUser({
		key: {
			providerId,
			providerUserId,
			password
		},
		attributes: {
			username
		} // expects `Lucia.DatabaseUserAttributes`
	});
} catch (e) {
	if (e instanceof LuciaError && e.message === `DUPLICATE_KEY_ID`) {
		// key already exists
	}
	// provided user attributes violates database rules (e.g. unique constraint)
	// or unexpected database errors
}
```

The fields of the `attributes` property is whatever you defined in `Lucia.DatabaseUserAttributes`. It should be an empty object with the default configuration (no user attributes defined).

You can optionally create a [key]() alongside the user. This is preferably compared to creating a key on its own as both the user and key will not be created if one fails (due to duplicate data etc). It will throw `AUTH_DUPLICATE_KEY_ID` if the key already exists. To not create a key, pass `null`:

```ts
await auth.createUser({
	key: null
	// ...
});
```

### User attributes errors

If the user attributes provided violates a database rule (such a unique constraint), Lucia will throw the database/driver/ORM error instead of a regular `LuciaError`. For example, if you're using Prisma, Lucia will throw a Prisma error.

## Update user attributes

You can update attributes of a user with [`Auth.updateUserAttributes()`](). You can update a single field or multiple fields. It returns the user of the updated user, or throws `AUTH_INVALID_USER_ID` if the user does not exist.

> (red) **Make sure to invalidate all sessions of the user on password or privilege level change.** You can create a new session to prevent the current user from being logged out.

```ts
import { auth } from "./lucia.js";
import { LuciaError } from "lucia";

try {
	const user = await auth.updateUserAttributes(
		userId,
		{
			username: newUsername
		} // expects partial `Lucia.DatabaseUserAttributes`
	);
} catch (e) {
	if (e instanceof LuciaError && e.message === `AUTH_INVALID_USER_ID`) {
		// invalid user id
	}
	// provided user attributes violates database rules (e.g. unique constraint)
	// or unexpected database errors
}
```

```ts
const user = await auth.updateUserAttributes(userId, {
	role: "admin" // new privileges
});
await auth.invalidateAllUserSessions(user.userId); // invalidate all user sessions => logout all sessions
const session = await auth.createSession(user.userId); // new session
// store new session
```

## Delete users

You can delete users with [`Auth.deleteUser()`](). All sessions and keys of the user will be deleted as well. This method will succeed regardless of the validity of the user id.

```ts
import { auth } from "./lucia.js";

await auth.deleteUser(userId);
```

## Configuration

You can configure users in a few ways:

- User id with [`generateUserId()`]()
- User attributes with [`getUserAttributes`]()