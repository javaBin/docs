# javaBin

## JavaZone

Active systems for JavaZone:

### Websites

Currently - https://github.com/javaBin/2025.javazone.no

Planned - https://github.com/javaBin/2025.javazone.no-native

### Mobile apps

Currently - https://github.com/javaBin/javazone-ios-app and https://github.com/javaBin/javazone-android-app

Planned - https://github.com/javaBin/2025.javazone.no-native

### Sleeping Pill

Sleeping Pill is the main data store for all JavaZone talks.

It is used by [Cake](#cake), [SubmitIt](#submitit), the [website](#wesites) and the [mobile apps](#mobile-apps) to provide the data for each year.

Current repository: https://github.com/javaBin/moresleep

### Cake

Cake is the interface used by pcom to evaluate and schedule CFPs for each year.

It allows searching, filtering and tagging of talks.

Uses [Sleeping Pill](#sleeping-pill) as data store.

Current repository: https://github.com/javaBin/cake-redux

### SubmitIt

This is the system that handles JavaZone call for papers each year.

Uses [Sleeping Pill](#sleeping-pill) as data store.

Current repository: https://github.com/javaBin/submitthethird

### Moosehead

This app provides signup for workshops and JavaZone kids.

Current repository: https://github.com/javaBin/mooseheadreborn

### Switcheroo

Provides the info screens at JavaZone.

Current repository: https://github.com/javaBin/switchredux

### Cupcake and Frosting

- under development - not yet in use \*

A simpler cake providing read-only access to SP for the regions.

Current repositories:

Backend - [cupcake](https://github.com/javaBin/cupcake)
Frontend - [frosting](https://github.com/javaBin/frosting)

### Session Search

- under development - not yet in use \*

Searchable history with links to videos (where available) for all data we have available back to 2007.

Current repository: https://github.com/javaBin/session_history

### Connections

The architecture diagram in mermaid is still in beta and displays strangely - hoping that this will improve :)

```mermaid
architecture-beta
    group aws(cloud)[AWS]
    group vercel(cloud)[Vercel]
    group mobile(internet)[Mobile Apps]

    service sleepingPill(server)[MoreSleep] in aws
    service database(database)[Postgres] in aws
    service submitIt(server)[SubmitTheThird] in aws
    service cake(server)[Cake Redux] in aws
    service cupcake(server)[Cupcake] in aws
    service frosting(server)[Frosting] in aws

    service web(server)[Web] in vercel

    service iOS(internet)[iOS] in mobile
    service android(internet)[Android] in mobile

    junction awsJunction in aws
    junction cakeJunction in aws

    database:T -- B:sleepingPill
    awsJunction:B -- T:sleepingPill
    submitIt:R -- L:awsJunction
    cakeJunction:L -- R:awsJunction
    cakeJunction:R -- L:cake
    cakeJunction:B -- T:cupcake
    cupcake:B -- T:frosting
    web:R -- L:sleepingPill
    iOS:L -- R:sleepingPill
    android:L -- R:sleepingPill
```

## javaBin

### Web

Current repository: https://github.com/javaBin/java.no
