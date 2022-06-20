# Syncit

## Overview:
<br>
A change data capture (CDC) windows service that synchronizes data from "Syneton/AdminIS" CRM to TAMTAM.

AdminIS by Syneton is a well-adopted CRM among accountants in BE. Unfortunately, it's a Desktop application that is often installed in the cloud (e.g CloudBizz), and accessed via tools such as TeamViewer, Citrix, etc.

AdminIS use SQL Anywhere database (version 12), which can be used as embedded DB in a local machine or as a DB server.

To synchronize data we decided to directly access the client's DB, capture changes, and forward them to Tamtam.
Waiting for a partnership with Syneton in this specific stage wasn't a relevant idea.

In the case of Deg&Partners the CRM was hosted in Cloudbizz. According to their technical support, we can't remotely access the DB. Instead, we can locally install a tool, in Deg Cloudbizz server, which will have read access to DB.

This solution is mainly developed to respond to Deg&Partner's needs, some queries and logic may later be reconsidered in case we decided to sell it to other fiduciaries.

Recently, Syneton offered an API to somehow access its client's Data. According to them, it must be adopted before June 2021 as an alternative to direct DB access's kind of solutions. We decided to consider that once we have more advanced contacts with them, and/or willing to sell our solution to other fiduciaries other than Deg&Parnters.


## Design:
<br>

### Big picture:
<img width="990" alt="Screen Shot 2021-04-23 at 11 26 02" src="https://user-images.githubusercontent.com/7085815/115863832-8eb67700-a425-11eb-8402-8b27d41882ca.png">

- An Aggregate can be an entity or a collection of entities with a root one, it's also simply interpreted as a resource.
- Sync DB is SQLITE file DB, it has two tables for snapshots and events.
- The term snapshot means an aggregate state signature based on the MD5 hash. It's used to detect whether the aggregate has changed or not since the last iteration.

In addition to CDC background service, another service is installed to manage updates of the first one:
- It polls a Tamtam AP to look for available updates.
- If a new version exists, it downloads it from an S3 bucket, installs it, then restarts the CDC service.

Note that the backup logic of sync DB is not yet implemented. (recently found an iteresting tool https://litestream.io/getting-started)

<br><br>
### Project structure:
<br>
The project has three modules:

- **Service** deals with the synchronization logic.
- **Tools** has the updater and installer CLI tool
- **Common** presents the library part i.e it contains a Tamtam OAuth client, and a homemade & simple dependency injection container.


<br>

The main module **Service** tries its best to respect a hexagonal architecture:

- **Domain** package holds the pure logic of capture and forward changes
- **Adminis** package contains the implementation of aggregates' fetchers. A fetcher is a kind of read-only Repository that connects to SQL Anywhere DB and fetches data.
- **Syncdb** package implements the store interface and manages the local SQLITE DB.
- **Application** offers the background service implementation for each use cases (e.g windows background service, CLI, etc)
- **main.go** the entry point.
<br><br>


### Processes:

We can identify 4 main processes:

- Capture data changes
- Forward changes to Tamtam API
- Update software
- Backup of sync DB (source of truth)
- Receive and process changes (in Tamtam API)
<br><br>


### Capture changes

 For the majority of resources Syncit will:

 1 - Fetches all available rows using a SQL query.

 2 -  Generates a signature of the recent fetched rows data using MD5 hash.

 3 - Compares the generated hash with the latest one saved in the resource snapshot (**AGGREGATE** table in SyncDB)  

 4 - the comparison may generate one of the 3 following CRUD events **ADDED, UPDATED, DELETED**

 5 - Both the new aggregate snapshot and the event are persisted in sync DB

 Note that:
  - We can pass a **SINCE** date parameter to fetchers of the resources that have a Date column e.g **created_at, updated_at** to avoid fetching all available rows.
  - A **SALT** property was introduced in some resources model to force a re-sync.
<br><br>

 ### Forward changes to Tamtam API

 The forwarder service fetches a chunk of events in each iteration from sync DB and sends it to Tamtam API.
 Note that:
 - The events chunk is removed from sync DB in case of a successful forward. Otherwise, The forwarder will keep trying until the transfer of the current chunk succeeds.
 - A **device key** is mandatory to communicate with Tamtam API.
<br><br>

### Receive and process changes

**Syncit** communicates only with Tamtam **Prod API** (v2). Prod API in turn forwards the same chunk to **RC2 API** for testing purposes.

(Note that the updater still use **DEV API**)

```sh
curl -X POST \
  https://api.tamtam.pro/adminis/receive-changes \
  -H 'Authorization: Bearer 2b2a4db80aaed9bd1bff7509883ed68fbb1fa4b3' \
  -H 'Content-Type: application/json' \
  -H 'Ttp-device-key: g10c9dcc0c44856bed96968256f126f26a76d5cc02961c587b8e3a55727bf197' \
  -H 'cache-control: no-cache' \
  -d '{
            "events": [
                ...
            ],
            "timestamp": "2019-12-12T18:03:56.2796149+01:00"
        }'
```

Event structure:

```JSON
{
  "event_id": 149227,
  "aggregate_id": "customer_id:10042",
  "aggregate_type": "society_customer",
  "type": "UPDATED",
  "hash": "71960397fed2053a998e5e9fc4b2640a",
  "timestamp": "1612950238",
  "data": {
    "Adress1": "Chemin d'Havré 70",
    "Adress2": null,
    "CRMTypeDenomination": "Klant",
    "CRMTypeID": "3",
    "City": "Saint-Symphorien",
    "Country": null,
    "EnterpriseNumber": "0762713869",
    "ID": "10042",
    "LanguageID": "2",
    "LegalFormID": "127",
    "Managers": [
      {
        "Function": "Administrateur(trice)",
        "ManagerEmail": "rastimeshin@icloud.com",
        "ManagerFirstName": "Valeri",
        "ManagerID": "9969",
        "ManagerLanguageID": "2",
        "ManagerLastName": "Rastimeshin",
        "SocietyID": "10042"
      }
    ],
    "Mobile": "+32 (494) 335743",
    "Name": "V.R. Dental",
    "Salt": "AZERTY",
    "Tel1": null,
    "Tel2": null,
    "Title": "SRL",
    "VatNumber": "NA",
    "Zipcode": "7030"
  }
}

```

**receive-changes AP** saves the received chunk in an S3 bucket first, then process its events using a set of handlers defined in https://github.com/Bruno-de-l-Escaille/ApiBundle/blob/v2/Resources/config/AdminIS/services.yaml#L3


In some cases e.g adding a new handler to receive resource changes in a new Tamtam project, we may need to replay the event stream from the beginning.

A CLI helper was developed to allow this:

https://github.com/Bruno-de-l-Escaille/ApiBundle/blob/v2/Command/AdminISReplayEventsCommand.php

<br>

## Local dev environment
<br>
AdminIS use an old version (12) of SQL anywhere which works well only in a windows environment. The easy way to locally test and run Syncit is to use a windows machine (or virtual machine).

1 - Install the SQL anywhere client. it's recommended to avoid the latest version, and choose the closest supported one to 12 (ex: 16)

2 - You have to install golang to be able to build exe files.

3 - Download a copy of Deg&partners DB (read-only access), and run it as SQLAnywhere server.

4 - You have to adapt credentials and some variables in the project code source to connect to the local DB and to mock forward events if needed.

5 - Once it's done, you have to build executables by running the following command in both **./Service** and **./Tools** folders:

```sh
go build -tags windows -ldflags="-X github.com/redaLaanait/syncit/common/tamtam.ClientID=30011 -X github.com/redaLaanait/syncit/common/tamtam.ClientSecret=a1326de3d89462026397f94b8ed3a5dxb22fb6c7"
```
<br>

## Installation guide (local and prod envs):
<br>
 1 - Open a powerShell console with administrator privileges (alt + f + s + a)
 
 2 - Run ``` .\tools.exe install-service``` command to install & start the change data capture handler as a windows service.
 
 3 - Run ``` .\tools.exe watch``` to install and start the updates manager service.
 
 
 4 -  other commands:
 
      * ``` .\tools.exe remove-service```
      * ``` .\tools.exe stop-watch```

 5 - the updates manager service isn't mandatory in the local env.

## Deg&Partners' Prod env
<br>
Cloudbizz give us an access to Deg's windows server

Credentials:

    username: rlaanait_DP2
    pwd: Cloud1****

Login link:

https://secure.cloudbizz.com/vpn/index.html

Note that **Two-factor Authentication** is enabled, so the user must receives a code per mail and/or SMS.

Once user is logged-in for the first time, cloudbizz web app will auto-install **Citrix** tool (like Teamviewer), and open a window session.


To open **AdminIS by Syneton** in Deg's server you need the following credentials:

    Login: Reda
    pwd: Lr0****
