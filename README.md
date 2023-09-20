# Powersync + Flutter

## Introduction

Trello App Clone built with [Flutter](https://flutter.dev/) :star2: [Powersync](https://powersync.co/) and [Supabase](https://supabase.io/)

![Banner of the images](showcase.png)

- `trelloappclone_powersync_client` is a library that defines the domain model and integrates with Powersync and Supabase
- `trelloappclone_flutter` contains the Trello clone Flutter app

## Getting Started

First checkout out [the getting started guide](https://docs.powersync.co/integration-guides/supabase-+-powersync) for [Powersync](https://powersync.co/) and [Supabase](https://supabase.io/).

TODO: describe below steps from basic tut.
- [ ] Create Powersync & Supabase accounts

## Configuring Supabase
TODO: describe below steps from basic tut.
- [ ] Setup Supabase project

### Creating Database Tables
After creating the Supabase project, we still need to create the tables in the database. 

TODO: use `tables.sql` and explain how to run this in Supabase SQL editor
 
- [ ] Create tables in Supabase (using generated sql files)
- [ ] set email confirmation flow to false

### Create the Postgres Publication

PowerSync uses the Postgres [Write Ahead Log (WAL)](https://www.postgresql.org/docs/current/wal-intro.html) to replicate data changes in order to keep PowerSync SDK clients up to date.

Run the below SQL statement in your Supabase SQL Editor:
```sql
create publication powersync for table activity, attachment, board, card, checklist, comment, listboard, member, trellouser, workspace;
```

## Configuring PowerSync

TODO: create below instructions from basic tut
- [ ] Setup Powersync project 
- [ ] and connect to Supabase

## Configuring Flutter App
TODO: describe below steps from basic tut.
- [ ] Configure Flutter app with powersync project settings (see basic Powersync Tut)

Note that at this stage _we have NOT yet setup any sync rules_. This means that the app will not yet retain any data that you create locally. What actually happens is that the data gets synced to Supabase (you can check with the Supabase SQL editor), but since there are no valid sync rules, Powersync deletes it from the local store, thus making it look like the workspaces and boards your created, are disappearing.

## Configuring Sync Rules
[Sync rules](https://docs.powersync.co/usage/sync-rules) are necessary so that PowerSync knows which data to sync to which client app instance.

### Global sync rules to get things working

We can be naive about it, and use a global bucket definition, that at least specify in some way which users can get data. 

- `trelloappclone_powersync_client.dart/sync-rules-0.yaml` provides such a naive approach. 
- Copy these sync rules to `sync-rules.yaml` under your Powersync project instance, and deploy the rules. 
- When you now run the app, it will actually show and retain data.
- The app code itself applies some basic filtering to only show data that belongs to the current user, or according to the visibility and membership settings of the various workspaces and boards.

### Using sync rules to enforce permissions
However, it is better that we do not sync data to the client that the logged-in user is not allowed to see. We can use Powersync sync rules to enforce permissions, so that users can only see and edit data that they are allowed to see and edit.

First, we need to understand the permissions from the app domain model:

- A **workspace** is created by a user — this user can always see and edit the workspace.
- A **workspace** has a specific *visibility*: private (only the owner can see it), workspace (only owner and members can see it), or public (anyone can see it).
- A **workspace** has a list of *members* (users) that can see and edit the workspace, if the workspace is not private.
- A **board** is created by a user — this user can always see and edit the board as long as the user can still access that workspace
- A **board** has a specific *visibility*: private (only the owner can see it), workspace (only owner and members belonging to the parent workspace can see it)
- A user can see (and edit) any of the **cards** and **lists** belonging to a **board** that they have access to.
- A user can see (and edit) any of the **checklists**, **comments**, and **attachments** belonging to a **card** that they have access to.

Also have a look at `trelloappclone_flutter/lib/utils/service.dart` for the access patterns used by the app code.

Let us explore how we can use sync rules to enforce these permissions and access patterns.

First we want to sync the relevant `trellouser` record for the logged-in user, based on the user identifier:

```yaml
bucket_definitions:
  user_info:
    # this allows syncing of trellouser record for logged-in user
    parameters: SELECT id as user_id FROM trellouser WHERE trellouser.id = token_parameters.user_id
    data:
      - SELECT * FROM trellouser WHERE trellouser.id = bucket.user_id
```

Then we want to look up all the workspaces (a) owned by this user, (b) where this user is a member, or (c) which are public.

```yaml
  by_workspace:
    # the entities are filtered by workspaceId, thus linked to the workspaces (a) owned by this user, (b) where this user is a member, or (c) which are public
    # Note: the quotes for "workspaceId" and "userId" is important, since otherwise postgres does not deal well with non-lowercase identifiers
    parameters:
      - SELECT id as workspace_id FROM workspace WHERE
        workspace."userId" = token_parameters.user_id
      - SELECT "workspaceId" as workspace_id FROM member WHERE
        member."userId" = token_parameters.user_id
      - SELECT id as workspace_id FROM workspace WHERE
        visibility = "Public"
    data:
      - SELECT * FROM workspace WHERE workspace.id = bucket.workspace_id
      - SELECT * FROM board WHERE board."workspaceId" = bucket.workspace_id
      - SELECT * FROM member WHERE member."workspaceId" = bucket.workspace_id
      - SELECT * FROM listboard WHERE listboard."workspaceId" = bucket.workspace_id
      - SELECT * FROM card WHERE card."workspaceId" = bucket.workspace_id
      - SELECT * FROM checklist WHERE checklist."workspaceId" = bucket.workspace_id
      - SELECT * FROM activity WHERE activity."workspaceId" = bucket.workspace_id
      - SELECT * FROM comment WHERE comment."workspaceId" = bucket.workspace_id
      - SELECT * FROM attachment WHERE attachment."workspaceId" = bucket.workspace_id
```

Have a look at `trelloappclone_powersync_client.dart/sync-rules-1.yaml` for the full set of rules; you can copy and paste this to `sync-rules.yaml` under your Powersync project instance, and deploy the rules.

## Build & Run the App

- Run ``` flutter pub get ``` to install the necessary packages on your command line that's navigated to the root of the project.
- Invoke the ``` flutter run ``` command, and select either an Android device/emulator or iOS device/simulator as destination (Powersync does not support flutter web app yet)

### Importing / Generating Data

TODO

## App Architecture

TODO: explain how it sticks together

1. overview of layers (app, client lib)
1. end-to-end discussion of how an entity is created, updated, watched, deleted
1. discussion of watch query example
1. discussion of transaction example?
1. discussion on IDs?

## Feedback

- Feel free to send feedback . Feature requests are always welcome. If there's anything you'd like to chat about, please don't hesitate to [reach out to us](https://docs.powersync.co/resources/contact-us).


## Acknowledgements

This tutorial is based on the [Serverpod + Flutter Tutorial](https://github.com/Mobterest/serverpod_flutter_tutorial) by [Mobterest](https://www.instagram.com/mobterest/

# TODOs

### Must DOs
- [X] Basic Powersync and Supabase setup
- [X] Create Supabase tables
- [X] Port Serverpod client code to Powersync client code
- [X] Update App code to user powersync client code
- [X] Test if it works with local db (sqlite)
- [X] Update to use Supabase Auth
- [X] implement basic global sync rules
- [X] Test if global syncing works
- [X] Tweak datamodel to allow per workspace lookups: add workspaceId reference to various entities (update on supabase, etc)
- [X] Tweak sync rules to enforce permissions (according to workspace owner, visibility, and members)
- [X] Test if permissions are enforced by Powersync
- [X] Look at using watch queries (for when other users update data)
- [X] BUG: look at listboard loading for new boards? (not seeing new empty lists..)
- [X] Data generation functionality for testing bigger datasets
- [X] Show message/spinner while syncing after login - check sync status (lastSyncedAt)
- [X] remember logged-in state (Supabase side remember auth session, but app not...)
- [ ] remove/hide/disable unused things (alternative logins, offline boards, my cards, members buttons)
- [ ] Look at using transactions (if time, else in next round))
- [ ] README/Tutorial writing & cleanup
- [ ] Clean up/disable non-working buttons and stuff

### Nice to haves?
- 
- [ ] (nice2have) get attachment uploads working with supabase
- [ ] (nice2have) make members functionality in app useful (currently only adding the owner as member to a workspace)
- [ ] (nice2have) implement email confirmation flow
- [ ] (nice2have) improve logging

### Bugs in app

Pre-existing bugs in the trello clone app.
Do we fix this or not?

- [ ] comments does not seem to work
- [ ] seems actual dragging of cards to different lists does not work?
- [ ] creation of checklist items are flaky...
- [ ] board details screen shows static data
- [ ] board share option cannot include "public" as an option

## Possible next steps

* Allow user to import existing workspaces/board/etc from e.g. Trello (either via API or exported json)
* Email confirmation flow
* Update of password or email
* Enhancing UX of app (there are many irritating issues and things not working yet in the original app)
* Enhance state management - e.g. let `TrelloProvider` listen to streams, and notify changes, to simplify views

## Changes from original Trello clone app

TODO: decide if we want to discuss this in the tutorial.

- Updated data model so that all `id` fields are Strings, and using UUIDs (it was auto-increment integer fields in the original app)
- Updated data model so that all entities refers to the `workspaceId` of workspace in which it was created (this facilitates the sync rules)