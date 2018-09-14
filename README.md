# Verso - Node.js LoopbackAPI-based storage middleware

## Introduction

_Verso_ is a boilerplate [Loopback API](https://loopback.io/) REST application for use with the Library of Congress' [BIBFRAME editor](https://github.com/lcnetdev/bfe) and its related [profile editor](https://github.com/lcnetdev/profile-edit). It offers endpoints for a few different kinds of storage:

1. `/verso/api/bfs`: Memory-based storage of BIBFRAME graphs for use with the BIBFRAME editor.
2. `/verso/api/configs`: Memory-based storage of JSON configuration objects for use with the profile editor.
3. `/verso/api/Users`: Memory-based storage of LoopBack User records. See [Authentication and authorization](#authentication-and-authorization) below.

You can explore the API using the Loopback API explorer at `/verso/explorer`. The models themselves are in the `common/models` directory.

_Verso_ can be configured to use [MongoDB](https://www.mongodb.com/) or a simple file to persist the memory-based storage. See [Configuring storage](#configuring-storage) below.

## Known issues

* [MongoDB storage corruption](https://github.com/lcnetdev/verso/issues/5)

Saving a record from the Profile Editor results in corruption of the record when using a MongoDB storage backend. This makes MongoDB storage unusable with the Profile Editor at present, except as read-only.

## Getting started

_Verso_ is a [Node.js](https://nodejs.org/) application designed to be built and run with [npm](https://npmjs.com). 

### Installation

Create a directory to hold your application, and navigate to that directory. Then:

```
git clone https://github.com/lcnetdev/verso
cd verso
npm install
```

### Configuring storage

_Verso_'s `db` datasource can be configured using the following environment variables:

`DB_STORAGE`: If set to `file`, the data for the `db` datasource will be stored and read from a JSON file generated by the `memory` connector. If set to `mongodb`, the data will be stored and read from a MongoDB instance. If this is not set, data will be stored in memory only, and not persisted between runs of the application.

`DB_FILE`: The path to the file for storage if `DB_STORAGE=file`. A file with sample BIBFRAME data (for the /verso/api/bfs endpoint) is included in this repository as `bfpilot.json`.

`DB_URL`: The MongoDB connection string if `DB_STORAGE=mongodb`. Format is `mongodb://[user:password@]host[:port]/db`.

You can configure these variables using a `.env` file in the root directory of the application. For example:

```
# Environment variables
DB_STORAGE=mongodb
DB_URL=mongodb://localhost:27017/test
```

### Running _Verso_

TL;DR: `npm start`.

The first time you start _Verso_, you will be prompted to create an `admin` user. This user can be used to create and manage other users (see [Authentication and authorization](#authentication-and-authorization) below for more details). For example:

```
$ npm start

> verso@2.0.0-SNAPSHOT start /opt/bibliomata/verso
> node .

/opt/bibliomata/verso/data/profiles not found!
Trying data/profiles
/opt/bibliomata/verso/data/vocabularies not found!
Trying data/vocabularies
Created 33 profile(s)
Created 1 vocabularies
Web server listening at: http://localhost:3001
Browse your REST API at http://localhost:3001/verso/explorer

No admin role present, assuming first run.
? Email for admin user? admin@example.org
? Password for admin user? [hidden]
? Confirm password for admin user? [hidden]
Admin user created.
```

The application runs on port 3001 by default. 

#### Production deployment

_Verso_ can be run using the [PM2 Process Manager](http://pm2.keymetrics.io/) for a more production-appropriate deployment.

**Important Caveat**: If you do this, you must _first_ configure storage and run using npm as above to create and persist the admin user, as pm2 will not allow for interactively creating the admin account.

To use pm2 to deploy _Verso_:

```
sudo npm install pm2 -g
sudo pm2 start server/server.js --name verso
```

To check on the status of pm2 applications:

```
sudo pm2 status
```

You should get a status screen (with a short uptime).

To save your pm2 runtime configuration:

```
sudo pm2 save 
```

This will save a copy of the PM2 config in `/root/.pm2/dump.pm2`

To restart the cluster with the dump file:

```
sudo pm2 resurrect
```

## Sample data
Sample configuration data for the profile editor is in the `data` directory. If there is no data in the data store, the sample data will be loaded on server start.

## Authentication and authorization

_Verso_ uses the built-in LoopBack mechanisms (Users, Roles, RoleMappings, and ACLs) to protect its endpoints. The `User` model has been extended to allow an administrative user to maintain user accounts and assign users to roles using the LoopBack web services API. Users and Roles are stored using the `db` datasource, so if you are using a MongoDB backend, the users will be persisted between runs of the application (see [Configuring storage](#configuring-storage) above).

For more information on the LoopBack security model, see the [LoopBack documentation](https://loopback.io/doc/en/lb3/Authentication-authorization-and-permissions.html).

The following roles and ACLs are set up by default:

  * `admin` role: users with this role have full access to all remote endpoints.
  * `profile_editor` role: users with this role have full access to the `configs` endpoint.
  * All authenticated users have full access to the `bfs` endpoint and read access to the `configs` endpoint.
  * All access is denied for unauthenticated users.

On application startup, a bootscript runs to set up the default roles and to extend the `User` model. If there is no admin user present, the bootscript will prompt to create one (see [Running _Verso_](#running-verso) above).

On login to _Verso_, a custom hook on the User `login` remote method creates two cookies:

* `access_token`: a signed, HTTP-only session cookie that contains the LoopBack authentication token. Custom middleware looks for this cookie with every request, so that the `Authorization` HTTP header is not required. This allows applications like the BIBFRAME Editor and the Profile Editor to use _Verso_ as an authentication layer without significant code changes.

* `current_user`: a session cookie that contains user identity information, as a JSON object. For example:

```
{
  "id": 1234,
  "username": "superuser",
  "roles": [
    "admin"
  ]
}
```

This cookie could be used by applications to present identity information on the page.

### User management

You can create and manage users using the `Users` web services API. For example, to create a user:

1. Log into _Verso_ as an administrative user

```
$ curl -w '\n' -D - -X POST -H "Content-Type: application/json" \
-d '{"username":"admin","password":"topsecret"}' \
http://localhost:3001/verso/api/Users/login

HTTP/1.1 200 OK
X-XSS-Protection: 1; mode=block
X-Frame-Options: DENY
X-Download-Options: noopen
X-Content-Type-Options: nosniff
Content-Type: application/json; charset=utf-8
Content-Length: 135
ETag: W/"87-nLgI/naEchBSDSJrT1iiVlBu4/Q"
Vary: Accept-Encoding
Date: Thu, 13 Sep 2018 03:40:14 GMT
Connection: keep-alive

{"id":"nCZX5oXWzLWY6UzgV0vqWzlSgxBBho7AKFn9cl4bCXZURPpMtwgXEmuW30zcFDCe","ttl":1209600,"created":"2018-09-13T03:40:14.810Z","userId":1}
```

The authorization token for use in the `Authorization` header of subsequent requests is returned in the `id` property of the response.

2. Create a user. Note that `email` is a required property.

```
$ curl -w '\n' -D - -X POST -H "Content-Type: application/json" \
-H "Authorization: nCZX5oXWzLWY6UzgV0vqWzlSgxBBho7AKFn9cl4bCXZURPpMtwgXEmuW30zcFDCe" \
-d '{"username":"new-user","password":"topsecret","email":"new-user@example.org"}' \
http://localhost:3001/verso/api/Users

HTTP/1.1 200 OK
X-XSS-Protection: 1; mode=block
X-Frame-Options: DENY
X-Download-Options: noopen
X-Content-Type-Options: nosniff
Content-Type: application/json; charset=utf-8
Content-Length: 61
ETag: W/"3d-X2TsLQGE0+X8tFjXyWQth8ndnro"
Vary: Accept-Encoding
Date: Thu, 13 Sep 2018 03:44:59 GMT
Connection: keep-alive

{"username":"new-user","email":"new-user@example.org","id":2}
```

This will create a user with full access to the `bfs` endpoint, and read access to the `configs` endpoint.

3. Add the user by ID to the `profile_editor` role.

```
$ curl -w '\n' -D - -X PUT \
-H "Authorization: nCZX5oXWzLWY6UzgV0vqWzlSgxBBho7AKFn9cl4bCXZURPpMtwgXEmuW30zcFDCe" \
http://localhost:3001/verso/api/Users/2/roles/profile_editor

HTTP/1.1 204 No Content
X-XSS-Protection: 1; mode=block
X-Frame-Options: DENY
X-Download-Options: noopen
X-Content-Type-Options: nosniff
Content-Type: application/json; charset=utf-8
Vary: Accept-Encoding
Date: Thu, 13 Sep 2018 03:49:37 GMT
Connection: keep-alive
```

This will grant the user full access to the `configs` endpoint.

You can explore the User API further using the Loopback API explorer at `/verso/explorer`.

### Disable authentication and authorization

If you want to work with _Verso_ without authentication and authorization for testing or development, you can disable it by removing the bootscript `server/boot/00-authentication.js`.

### Test users

If the environment variable `DEV_USER_PW` is set, the users `admin`, `profile_editor`, and `user` will be created with the password set to the value in the variable.
