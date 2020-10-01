# UIS-in-nodejs

### A node.js based hack to access Boston College's student course registration system (i.e. UIS).

## Note: This repo houses only the readme for the package. Due to concerns with security, the original repo is private. If you want access please contact me separately.  

## Motivation

Boston College's University Information System (UIS) is a IBM AS/400 mainframe that handles course registration and administrative work.
The system exposes a telnet connection accessible by a [tn3270](https://en.wikipedia.org/wiki/IBM_3270#Telnet_3270) capable terminal emulator for interaction.
This means that students are limited to a [Text User Interface (TUI)](https://en.wikipedia.org/wiki/Text-based_user_interface) to register for courses.

While this isn't a problem per se, it does however mean that the system is locked for development for any developers that don't know telnet, IBM 3270, and the [EBCDIC](https://en.wikipedia.org/wiki/EBCDIC) encoding.

This project hopes to alleviate the dependency on a protocol with a steep learning curve and allow for creative access of UIS through a more modern interface: nodejs.

- [Motivation](#Motivation)
- [Mechanism](#Mechanism)
- [References](#References)
- [Quickstart](#Quickstart)
- [API](#api)
  - [UISSession](<#UISSession-`new-UISSession(sessionId[,-disableLog])`>)
    - [Properties](#properties)
      - [.courses](#uissession.courses)
      - [.disableLog](#uissession.disableLog)
      - [.loggedIn](#uissession.loggedin)
      - [.logger](#uissession.logger)
    - [Methods](#methods)
      - [.login()](<#`async`-uissession.login(username,-password[,-accesscode])>)
      - [.logout()](<#`async`-uissession.logout()>)
      - [.modify()](<#`async`-uissession.modify(regRequest)>)
      - [.save()](<#`async`-uissession.save()>)
      - [.undo()](<#`async`-uissession.undo()>)
    - [Types](#types)
      - [Course](#course)
      - [CourseRegRequest](#CourseRegRequest)
    - [Events](#events)
      - ['ERROR'](#ERROR)
  - [UISSession.logger](#Logger-`UISSession.logger`)
    - [Properties](#properties-1)
      - [.logFile](#Logger.logFile)
      - [.disable](#Logger.disable)
      - [.sessionId](#Logger.sessionId)
    - [Methods](#methods-1)
      - [.log](<#logger.log(stream)>)
      - [.logScreen](<#Logger.logScreen(rows,-direction)>)
      - [.logError](<#Logger.logError(error)>)
    - [Types](#types-1)
      - [Direction](#`enum`-direction)
- [Error Handling](#Error-handling)
  - [UISError](#uiserror)
    - [Properties](#properties-2)
  - [Error Codes](#Error-codes)
- [Todo](#Todo)

## Mechanism

UIS-in-nodejs is a middleman of sorts that handles the tn3270 connection over TCP/IP from UIS and exposes methods that are easier to work with.

## References

Based most of my code from this repo and adopted it for UIS.

- https://github.com/mflorence99/tn3270

A reference of telnet commands

- http://users.cs.cf.ac.uk/Dave.Marshall/Internet/node141.html

A detailed look at tn3270

- http://www.tommysprinkle.com/mvs/P3270/

Another detailed look at tn3270

- http://www.prycroft6.com.au/misc/3270.html

Wikipedia for EBCDIC encoding

- https://en.wikipedia.org/wiki/IBM_3270#3270_character_set

# **Important: UIS can only be accessed on the Boston College network or through EagleVPN. Consequently, UIS-in-nodejs can only connect to UIS when the host device is connected to either the BC network or EagleVPN**

## Quickstart

Install dependencies:

```bash
npm install
```

Run code:

```javascript
let UISSession = require("./build/main.js");

let session = new UISSession(sessionId);

session.login(username, password).then(() => {
  let regRequest = {
    order: 0,
    course: 4442
  };

  session.modify(regRequest).then(() => {
    session.save().then(() => {
      session.logout();
    });
  });
});
```

# API

## UISSession `new UISSession([sessionId])`

- sessionId (optional): `<string | number>`
  - default: `a uuid v1`

Returns a new instance of `UISSession`.
Creating an instance does not connect to UIS.
Connection is created when `UISSession.connect()` or `UISSession.login(...)` is called.

### Properties:

#### UISSession.courses

- (Read-only) [`<Course[]>`](#Course)

An Array of [Course](#Course) objects. `undefined` before login.

#### UISSession.loggedIn

- (Read-only) `<boolean>`

A boolean indicating whether a user is logged in.

#### UISSession.menu

- (Read-only) `<string>`

A string indicating the name of the current navigated menu

### Methods:

#### `async` UISSession.login(username, password)

- username: `<string>`
- password: `<string>`
- Returns: `Promise<void>`

Method to login a student. Username and password required.

#### `async` UISSession.logout()

- Returns: `Promise<void>`

Method to logout. Doesn't actually logout per se, but destroys connection to UIS and therefore implicitly logs out. Does not update any changes to registration.

#### `async` UISSession.getToRegistration([accessCode])

- accessCode (optional): `<string | number>`
- Returns: `Promise<void>`

Method to navigate from after login to the registration menu. If access code is required to navigate, will need access code or else will throw an error.

#### `async` UISSession.loginAndGetToRegistration(username, password[, accessCode])

- username: `<string>`
- password: `<string>`
- accessCode (optional): `<string | number>`

Method that calls `UISSession.login(...)` then `UISSession.getToRegistration(...)`.

#### `async` UISSession.modify(regRequest)

- regRequest: [`<CourseRegRequest | CourseRegRequest[]>`](#CourseRegRequest)
- Returns: [`Promise<Course[]>`](#course)

Method to modify course registration. Takes a [CourseRegRequest](#CourseRegRequest) object or an array of such objects as parameters. It updates the student's registration. It does not save the registration.

_**Important** note about registration:_
If there is an issue with the registration, an error won't be thrown but the `alert` property will contain the alert.
UISSession will not save if there are unresolved alerts

#### `async` UISSession.save()

- Returns: `Promise<void>`

Method to save. UISSession will not save if there are unresolved alerts. If there are unresolved alerts, errors will be thrown.

_**Important** note about errors:_
If there are unresolved alerts from a modification request (ie UISSession.modify(...)), they will be thrown as errors here. If there is one alert, an `Error 3xx` will be thrown. If there are multiple alerts, an `Error 300` will be thrown and each individual alert can be found in `UISError.comments.alerts`.

#### `async` UISSession.undo()

- Returns: `Promise<void>`

Method to undo everything. Will undo any changes after the latest save or since login.

### Types:

#### Course

```typescript
type Course = {
  order: number;
  index: number;
  code: string;
  alert?: string;
};
```

#### CourseRegRequest

```typescript
type CourseRegRequest = {
  order: number;
  course: number | string;
};
```

The course property is either a number representing the course Index or a string representing the course code. The string must match the following regular expression to be valid:

> `/\d{4}|[a-zA-Z]{4}\d{6}/i`
> Four character departemnt code + 4 digit course code + 2 digit section number.

### Events:

#### 'ERROR'

<!-- TODO -->

- exception: [`<UISError>`](#UISError)

`'ERROR'` is emitted when any error is thrown outside of method calls. Such as when the connection is closed by UIS, and when UIS logs out a student for inactivity.

#### 'BIRTHDAY'

<!-- TODO -->

- `<void>`

`'BIRTHDAY'` is emitted when UIS gives a birthday message to the student.

#### "LOGGED_OUT"

<!-- TODO -->

#### "OFFLINE"

<!-- TODO -->

# Error Handling

Due to the spaghetti code, errors that result directly from method calls are thrown normally, but errors that result from anywhere else are emitted as an "ERROR" event by UISSession.

> might refactor idk...

## UISError

### Properties:

- code: `<number>`
- name: `<string>`
- message: `<string>`
- comments: `<{[index: string]: any}>`

Error codes are listed [below](#Error-Codes). `UISError.name` and `UISError.message` are the corresponding names and messages.

Errors thrown during a `UISSession.modify(...)` call will have the offending course in the comments at `UISError.comments.course`. If it is `Error 300`, there will be specific information found at `UISError.comments.problems`

## Error Codes

<!-- TODO update error codes list -->

| Code | Name                             | Message                                                                                                                                                                                                                                     |
| ---- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 100  | UIS_NOT_REACHABLE                | uis is unable to be connected                                                                                                                                                                                                               |
| 101  | UIS_OFFLINE                      | uis is offline for the night                                                                                                                                                                                                                |
| 102  | NAVIGATION_ERROR                 | navigated to unknown menu                                                                                                                                                                                                                   |
| 103  | UNKNOWN_PROCEDURE_AT_MENU        | attempted to navigate at an unknown menu                                                                                                                                                                                                    |
| 104  | LOGIN_INVALID_MENU               | attempted to login on not a login menu                                                                                                                                                                                                      |
| 105  | ACCESS_CODE                      | access code required                                                                                                                                                                                                                        |
| 106  | NOT_CONFIRMATION_MENU_AFTER_SAVE | navigated not to confirmation menu after saving course                                                                                                                                                                                      |
| 107  | RESPONSE_TIMEOUT                 | host response from request timed out                                                                                                                                                                                                        |
| 108  | INCORRECT_ACCESS_CODE            | access code incorrect                                                                                                                                                                                                                       |
| 109  | COURSE_MENU_REQUIRED             | must be on course menu to do action                                                                                                                                                                                                         |
| 110  | REGISTRATION_ENDED               | registration ended                                                                                                                                                                                                                          |
| 200  | UNAUTHENTICATED                  | not logged in                                                                                                                                                                                                                               |
| 201  | LOGIN_ERROR                      | incorrect username or password                                                                                                                                                                                                              |
| 202  | LOGIN_ELSEWHERE                  | user logged in on different session                                                                                                                                                                                                         |
| 203  | USERNAME_LENGTH                  | entered username is too long                                                                                                                                                                                                                |
| 204  | INVALID_USERNAME                 | username is invalid                                                                                                                                                                                                                         |
| 205  | INACTIVE_USERNAME                | username is inactive                                                                                                                                                                                                                        |
| 206  | INCORRECT_PASSWORD               | incorrect password                                                                                                                                                                                                                          |
| 207  | ENTER_USERNAME                   | enter username, no username given                                                                                                                                                                                                           |
| 208  | NO_PASSWORD                      | enter password, no password given                                                                                                                                                                                                           |
| 300  | MULTIPLE_ERRORS                  | multiple errors: see comments                                                                                                                                                                                                               |
| 301  | INVALID_INDEX                    | incorrect index                                                                                                                                                                                                                             |
| 302  | INVALID_COURSE                   | invalid course                                                                                                                                                                                                                              |
| 303  | TIME_CONFLICT                    | course has a time conflict with another course                                                                                                                                                                                              |
| 304  | COURSE_CODE_VALIDATION_FAIL      | invalid course code format. course code must either be a up to 4 digit index (ie. max 9999) or 4 character department code followed by the 4 digit course number followed by the 2 digit section number(ie ECON123456). is case insensitive |
| 305  | NEEDS_DEPARTMENT_PERMISSION      | need department permission to register for this course                                                                                                                                                                                      |
| 306  | COURSE_CLOSED                    | course closed                                                                                                                                                                                                                               |
| 307  | NO_COURSES_READ                  | no courses received from UIS                                                                                                                                                                                                                |
| 399  | MISC_COURSE_REGISTRATION_ERROR   | check comments for details                                                                                                                                                                                                                  |
| 400  | UNKNOWN_JS3270_ERROR             | unidentified error from tn3270 process                                                                                                                                                                                                      |

# Todo:

- Identify all UIS Course Errors
- Update Errors.ts with UIS Course Errors
- Test UIS Course Errors
- Methods for access code input
- Methods and errors for when accessing too early
- Finish writing tests for Session.ts
- Write tests for main.ts
- Refactor menu recognition because its unruly
