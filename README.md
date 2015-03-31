# ScalableWebArchitectures

This Repository stores my projects for the Scalable Web Architectures class at University of Applied Science.

## How to run

This distributed system uses Rack. The systems are combined to one system in the config.ru file, which is 9292. 

To start services use the following commands:
```sh
	rackup -o 0.0.0.0 config.ru
```

## How to run the tests
To run the automated tests with Rspec, please start the server first. And restart the server after each call of the Rspec command
After many hours of trying to test the ReportSystem i decided to send its requests with Ruby's "system" command to ensure that the requests arrive correclty.
- Start the server in a Terminal window with the command above
- Type "rspec" in another Terminal window in the root folder to test with Rspec

## The system

This distributed system consists of four subsystems. These subsystems communicate via HTTP with each other.
The following figure shows the systems and their endpoints:

![Overview over all four systems](assets/final-challenge-systems-overview.png)
Source: https://github.com/nesQuick/2015-salzburg-soa-workshop/blob/master/challenges/final/assets/final-challenge-systems-overview.png

### The subsystems

#### User Management

The user management in our example is a very basic authentication system with 3 hard coded users. There is no way to add, change or delete users at all.

The system provides a single HTTP endpoint:

**GET /user**

All requests to that endpoint must be made using HTTP Basic Auth to authenticate as one of these three users:

User  | Password
----- | -------------
wanda | partyhard2000
paul  | thepanther
anne  | flytothemoon

If the authentication succeeds, that is the HTTP Basic Auth combination of user name and password is correct, the endpoint returns a status code ``200``. In any other case (user not found, password incorrect) the endpoint returns HTTP status ``403``. The response body must always be empty.

The user management system is never used from an end-user. Instead services query it to check if the authentication they get from the end-user is valid. E.g. if a user wants to pull a report from the report system she is authenticating her request to the report system with HTTP Basic Auth. The report system then calls the ``/user`` endpoint of the user management system and only if that returns a ``200`` status code the report is created. If the user system would return a ``403`` the report system would not create a report and instead return a ``403`` status code as well. [This diagram](assets/authentication.pdf) illustrates the flow (please see the split of events depending of a valid or invalid authentication in 3a/b and 4a/b).

#### Location Management System

The location management system holds a very simple model of data about each location where the company keeps stuff (e.g. their offices, warehouses, etc). The data model looks like this:

* name (any string)
* address (any string)
* id (an auto incremented integer that is set by the system)

So a location could look like this:

```ruby
{
  "name" => "Office Alexanderstraße",
  "address" => "Alexanderstraße 45, 33853 Bielefeld, Germany",
  "id" => 562
}
```

The system has actions to create, delete and list all locations. Each request must be authenticated with a valid user name / password combination (see User Management System for all existing users).

**POST /locations**

The request body must be a JSON (see Ruby's JSON library and check ``JSON.parse`` and ``JSON.dump``) encoded location object without the id like this:

```json
{
  "name": "Office Alexanderstraße",
  "address": "Alexanderstraße 45, 33853 Bielefeld, Germany"
}
```

The system should create that record internally, and return a HTTP status ``201`` with a complete JSON representation of the location (including the ID) as it's response body. Like this:

```json
{
  "name": "Office Alexanderstraße",
  "address": "Alexanderstraße 45, 33853 Bielefeld, Germany",
  "id": 562
}
```

If not all fields are send in the request return an HTTP error code and a JSON encoded error message as the body.

**Important:** Please note that you do not need to use any data persistence (DB, Redis, …) for this challenge. Just keeping all the records in memory and therefore losing them whenever an app is terminated is good enough!

**GET /locations**

Requests shall have an empty body. Returns status ``200`` and a JSON encoded array of all locations known to the system. Like this:

```json
[
  {
    "name": "Office Alexanderstraße",
    "address": "Alexanderstraße 45, 33853 Bielefeld, Germany",
    "id": 562
  },
  {
    "name": "Warehouse Hamburg",
    "address": "Gewerbestraße 1, 21035 Hamburg, Germany",
    "id": 563
  },
  {
    "name": "Headquarters Salzburg",
    "address": "Mozart Gasserl 4, 13371 Salzburg, Austria",
    "id": 568
  }
]
```

**DELETE /locations/:id**

Requests shall have an empty body. Deletes the location specified by the ID supplied in the URL. It then returns status ``200`` and an empty body.

Returns status ``404`` if the supplied ID does not exist.

#### Item Tracking System

The item tracking system holds a very simple model of data about each tracked item. The data model looks like this:

* name (any string)
* location id (an integer referencing a location)
* id (an auto incremented integer that is set by the system)

So a tracked item could look like this:

```ruby
{
  "name" => "Johannas PC",
  "location" => 123,
  "id" => 456
}
```

The system has actions to create, delete and list all tracked items. Each request must be authenticated with a valid user name / password combination (see User Management System for all existing users).

**POST /items**

The request body must be a JSON encoded item object without the id like this:

```json
{
  "name": "Johannas PC",
  "location": 123
}
```

The system should create that record internally, and return a HTTP status ``201`` with a complete JSON representation of the tracked item (including the ID) as it's response body. Like this:

```json
{
  "name": "Johannas PC",
  "location": 123,
  "id": 456
}
```

If not all fields are send in the request return an HTTP error code and a JSON encoded error message as the body. The system does **not** have to check for the validity of the supplied location ids!

**GET /items**

Requests shall have an empty body. Returns status ``200`` and a JSON encoded array of all tracked items known to the system. Like this:

```json
[
  {
    "name": "Johannas PC",
    "location": 123,
    "id": 456
  },
  {
    "name": "Johannas desk",
    "location": 123,
    "id": 457
  },
  {
    "name": "Lobby chair #1",
    "location": 729,
    "id": 501
  }
]
```

**DELETE /items/:id**

Requests made with an empty body. Deletes the tracked item specified by the ID supplied in the URL. It then returns status ``200`` and an empty body.

Returns status ``404`` if the supplied ID does not exist.

#### Report System

The report system combines data from the location and item tracking system. It does not hold any data itself.

The system has only one endpoint to get a list of all tracked items grouped by their respective location. Each request must be authenticated with a valid user name / password combination (see User Management System for all existing users).

**GET /reports/by-location**

Requests shall have an empty body. Returns status ``200`` and a JSON encoded array of all locations. Each location object has a key ``items`` which holds an array of all items in that location. Like this:

```json
[
  {
    "name": "Office Alexanderstraße",
    "address": "Alexanderstraße 45, 33853 Bielefeld, Germany",
    "id": 562,
    "items": [
      {
        "name": "Johannas PC",
        "location": 562,
        "id": 456
      },
      {
        "name": "Johannas desk",
        "location": 562,
        "id": 457
      }
    ]
  },
  {
    "name": "Warehouse Hamburg",
    "address": "Gewerbestraße 1, 21035 Hamburg, Germany",
    "id": 563,
    "items": [
      {
        "name": "Lobby chair #1",
        "location": 563,
        "id": 501
      }
    ]
  }
]
```

## 12 Factor
This system obeys some of the principles of the 12 Factor Manifesto. Some of them didn't make sense, because it's too small.
Some of the principles are:
- Codebase
- Dependencies
- Processes
- Port binding
- ...
