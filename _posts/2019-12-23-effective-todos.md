---
layout: post
title: Writing Effective Todo Comments
author: 
    name: David Marland
    twitter: djmarland
image: /assets/effective-todos/banner.svg
---

“Todo” comments are inevitable part of development. Not all problems can be solved right away and so you find yourself stating `//todo` in a hope that you’ll come back to it.
Months or years later you encounter the line `// todo – fix this` but don’t know how. You reach for `git blame` only to find that you yourself wrote it. “Curse you, past me!”

It’s important to write todo comments in a way that’ll be most effective in getting them removed later. No todo should be immortal. These are some practices I try to follow.

## Syntax
Always begin the comment with `// todo – `. Jetbrains IDEs such as IntelliJ, PHPStorm and Webstorm recognise this syntax and will highlight the comment differently to a regular comment. If the todo spans multiple lines indent the following lines with at least one space and it will be considered all one
  
[![Screenshot of the todo comment Highlighting in Jetbrains IDEs](/assets/effective-todos/structure.png)](https://tubealert.co.uk)  
  
The benefit of following a structure the IDE recognises is the tools it can provide. Jetbrains offers a handy window listing your entire project’s todos. Open it from the View menu:
`View -> Tool Windows -> Todo`
 
[![Screenshot of the Todo finder in Jetbrains IDEs](/assets/effective-todos/navigator.png)](https://tubealert.co.uk)

This is a useful way of seeing tech debt all together and gives a good indication of the work that needs to be done.

## Work in Progress
Todos don't just have to be unknowns to solve some other time. They can be useful during single feature development too. Writing todos inline with your code can be a good reminder to know when your feature is ready.

```
// todo - before merge: check the user has permissions to do this
```

You may be working on a feature that requires a user permission check, but want to work on the main bit first. It can be easy to forget to add that restriction and then end up on production without it. By adding a temporary todo, you will see that you have yet to finish when you check your diff before merging or self-reviewing your PR (you do self-review right?). At the very least a colleague should see the comment and ask if you forgot to do this, where they would otherwise have not noticed the missing functionality.

## Linking Comments
Most todos are designed to be killed. Quite often it'll be temporary code that needs refactoring or removing later. But sometimes that may involve touching multiple files. How do you remember the multiple locations that need changing when you come to fix the todo later?

You can link todos together with a searchable string, sort-of a hashtag:

```php
// todo - #php7upgrade - This line can used the new spaceship operator
```

The example above shows how code could be improved later, after an upgrade. There could be many comments like this throughout the code in different files. By using the tag `#php7upgrade` they become linked. Now, after the upgrade you can find all related todos to fix.

When reading code if you encounter a todo beginning with one of these hashtags, you know that there must also be a related task somewhere else in the codebase and it is easily findable with a global search.

## Block todos
Sometimes code is written as a temporary shim or polyfill, to get through a phase of development. The cleanup effort later can be as simple as deleting the temporary code; but how do you know later on which code to delete if you encounter this:

```javascript
let users = getUsers();
// todo - delete this once all users have been migrated to the new format
const legacyUsers = getLegacyUsers();
const converted = convertUsers(legacyUsers);
users = users.concat(converted);
notifyUsers(users);
```

In the above code how do you know which lines are to be removed once all the users have been migrated. You can instead format the todo to wrap the code. A possible syntax to indicate this could be as follows:

```javascript
let users = getUsers();
// todo - delete this once all users have been migrated to the new format
// todo - <<<DELETE FROM HERE
const legacyUsers = getLegacyUsers();
const converted = convertUsers(legacyUsers);
users = users.concat(converted);
// todo - >>>DELETE TO HERE

notifyUsers(users);
```

As a developer you will come to recognise that the presence of `<<<DELETE` means there will be a corresponding `>>>DELETE` later in the code, making it much simpler to remove the legacy code. This syntax should of course only be used if the block can be deleted wholesale without breaking the remaining code.

Remember, there may be tests that would break when deleting this block, so it could be combined with the hashtag to know exactly what to delete.

```javascript
// todo - #legacyUsers <<<DELETE FROM HERE
const legacyUsers = getLegacyUsers();
...
// todo - #legacyUsers >>>DELETE TO HERE
```

```javascript
// todo - #legacyUsers <<<DELETE FROM HERE
it('should convert legacy users' () => {
...
});
// todo - #legacyUsers >>>DELETE TO HERE
```

So that's a few tips for writing effective todos. Obviously you can choose whichever syntax works for you, but if you remain consistent you will understand them weeks, months or even years in the future.
