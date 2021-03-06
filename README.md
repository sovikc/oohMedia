# Inventory
An inventory of physical display panels installed in shopping centres, airports, gyms, and highways.

## Approach
A mix of ideas from Domain-driven design and Uncle Bob's Clean Architecture implemented in TypeScript

## Low-level design
![alt text](https://github.com/sovikc/oohMedia/blob/master/low_level_design.png)

## Naming convention
1. The top-level directories **ims** and **db** have short meaningful names for devs to refer easily during daily conversations.
2. The modules in the sourcecode have domain-specific names so that the domain vocabulary is well-represented, such as inventory (housing domain entities), management (service to manage inventories of centres, assets etc.).
3. The infrastructure specific names are used to give the reviewers an idea what they can expect in the directories, like **postgres** would probably be expected to have SQL related files, whereas **routes** might have authentication middleware and handlers for the routes.

## Design Patterns used
1. **Dependency Injection** - Used to inject Repository implementation to management services and ID Generator into domain entities
2. **Factory** - Used to create domain entities
3. **Adapter** - Used in routes/handler to adapt the services to http request/response

## Code structure and directory layout
1. **inventory** - domain entities like Centre, Location, and Asset. A Centre denotes a shopping centre, or a mall. An Asset represents a physical panel located in a certain Location inside the centre. It also has the Repository and ID generation interfaces, Error declaration and factory functions for the entities and validators. 
2. **management** - domain services for managing centres and assets along with their allocation and deallocation.
3. **postgres** - database related code.
4. **id** - code for ID generation.
5. **routes** - route handling, request/response adapter, and user authentication middleware.
6. **controller** - transformation related code for request/response and invocation of services. 
7. **db/pg.sql** - ddl statements for the tables: shopping_centre, location_within_centre, asset, asset_allocation, and change_log.

## Database side of things
1. The database is designed as an OLTP database. It has following tables.
```
shopping_centre
location_within_centre
asset
asset_allocation 
change_log
```
2. All `insert` and `update` statements use database transactions with an isolation level of `Read Committed`.
3. All `deletes` are `soft` deletes.
4. Every data manipulation is recorded in `change_log` table and can be used for both `audit` as well as `event source` if the services need to move in that direction. 
5. All the connection parameters are stored in `.env` file.

## Rules
There are a 2 `entities` and a `relation`. These enities are `Shopping Centres` (called just Centres to avoid long names) and `Assets` and their relation being the `allocation of an Asset to a location inside a Shopping Centre`.

### Shopping Centre
1. A Centre object has a name and an address. The name is mandatory and unique so are all the address fields lineOne, city, state, postalCode, and country, with only exception being lineTwo of the address.
2. A Centre can have multiple Locations with unique codes to signify a location where an Asset can be installed. These locations can only be created once a Shopping Centre exists.
3. Removing a location also removes the allocations to that location.
4. Removing a Centre also removes the locations and allocations to those locations.

### Asset and its allocation
1. An Asset object has a name, an active status, length, breadth, and depth, and an optional allocation (to denote where it's allocated) since an Asset can exist without being allocated. 
2. An Asset cannot be allocated to a deactivated Centre or a location (inside a Centre) already occupied by another Asset, unless the occupying Asset is either deallocated or it's made inactive.
3. An Asset cannot be allocated again unless it's current allocation is removed. The UI can call 2 APIs and also inform/warn the user about what is going on behind the scene. 
4. If an Asset is not active anymore(due to being removed for maintenance or such, using the `PATCH /assets/:id`), then it's allocation to a Shopping Centre's location is automatically removed. 

## Tests
1. Jest is used as a testing framework.
2. Entity creation rules in the inventory have been tested with centre.spec.ts and asset.spec.ts.
3. Application rules in management services are covered by services.spec.ts.

## What's not done
1. APIs for user registration and login.
2. APIs for searching for Assets
3. User interface for the application
4. Docker secrets in the docker-compose files

## How to run
1. The code is dockerized and can easily be run with docker-compose. The docker-compose.yml has references to Dockerfiles in **ims** and **db**. It would be easier to use this file with `docker-compose build` followed by `docker-compose up` and also `docker-compose down` to remove the containers. 
2. The server will run on port 8080 and the request needs 2 headers `auth-token` with value `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6ImNrZjg2cm9sNzAwMDF6MTBmYjNveTNwaDUiLCJpYXQiOjE2MDA1OTY3MTB9.bbGI82--q4U9WIdn4KhAHuVlK4XpkG0moKm6lUPWEww` and `content-type`with value `application/json`. The `auth-token` will ensure that these APIs are protected against anonymous access.
3. The code has following APIs that can be used. There are example requests for `POST` and `PATCH` APIs.
```
   GET          /
   
   POST         /centres
   {
    "name": "Goldolfer",
    "address": {
      "lineOne": "500 Moore Street",
      "city": "Lancabin",
      "state": "Lower Brocci",
      "postCode": "500023",
      "country": "Isle of Grand Fenwick"
      }
    }

   GET          /centres/:id

   PATCH        /centres/:id
   {
   "address": {
      "lineTwo": "Cnr of Manning and Addison"
      }
   }

   DELETE       /centres/:id

   POST         /centres/:id/locations
   {
    "code": "L234"
   }

   PATCH        /centres/:id/locations/:locationId
   {
    "code": "5L1234"
   }

   DELETE       /centres/:id/locations/:locationId

   POST         /assets
   {
    "name": "Sign-01-1230-1050-0200",
    "length": 12.30,
    "breadth": 10.5,
    "depth": 2,
    "active": true
   }

   PATCH        /assets/:id
   {
    "active": false
   }

   POST         /assets/:id/allocate
   {
    "centreId" : "ckfi984yc0000ve0f8mpo9qjs",
    "locationCode": "L234"
   }

   DELETE       /assets/:id/allocate
```

