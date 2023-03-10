---
title: "Creating a pre-seeded sql docker image"
date: 2023-01-13T21:07:19+02:00
tags: [docker]
draft: false
---

### Introduction

On our company we wanted to create an easy way for someone to run our web application (ASP.NET Core backend + Blazor frontend)
through docker compose for demo/qa purposes, but there was a tiny problem.

Our web application uses an in-house authentication server docker image that
has a lengthy registration process. When someone uses the demo script for the first time or the
database container is deleted when a new version is released, he has to waste several minutes in order to
register our local application with the authentication server.

### The Idea

What if the server seeded the db when it ran?

But when will the seeding take place? 
We cannot change the code of the other team!
There is another... point, that can be used to inject logic;
The entrypoint of the docker image!

### Execution

We just have to create a bash script that will include our seeding logic and then create another
dockerfile that has the original docker image as base but will have the special entrypoint that will 
execute the below script instead.

#### entrypoint.sh

```bash
#!/bin/bash

if [ ! -f "/tmp/.data-seeded" ]; then
  # Expose sql credentials from connection string as variables
  for section in $(echo $ConnectionStrings__DefaultConnection | sed 's/ //g' | tr ";" "\n"); do
    export $(echo $section | sed 's/ //g')
  done

  echo 'INFO : Starting migrations and seeding!' &&
    dotnet Web.Server.dll --run-migrations &&
    /opt/mssql-tools/bin/sqlcmd -S $DataSource -U $UserID -P $Password -d $InitialCatalog -i /App/seed.sql &&
    echo 'INFO : Finished migrations and seeding!' &&
    echo '1' >/tmp/.data-seeded
fi

if [ ! -f "/tmp/.data-seeded" ]; then
  echo '#####'
  echo 'ERROR : Data was not seeded! this should not happen, contact dev team'
  echo '#####'
  exit 1
fi

dotnet Web.Server.dll
```

#### dockerfile

```dockerfile
FROM *REDACTED*.azurecr.io/*REDACTED*

COPY ["./seed.sql","."]
COPY ["./entrypoint.sh","."]

# MSSQL Tools
ADD https://packages.microsoft.com/keys/microsoft.asc /tmp/ms-keys
ADD https://packages.microsoft.com/config/ubuntu/20.04/prod.list /tmp/ms-repos

# Install necessary tools in order for sql tools to work
RUN apt-get update && apt-get install -y gnupg2 && \
  cat /tmp/ms-keys | apt-key add - && \
  cat /tmp/ms-repos | tee /etc/apt/sources.list.d/msprod.list && \
  apt-get update && ACCEPT_EULA=Y apt-get install -y mssql-tools unixodbc-dev

EXPOSE 80
EXPOSE 443

ENTRYPOINT [ "/bin/bash","./entrypoint.sh" ]
```

### Considerations

The downside of the above
idea is that instead of just using the image of the external service, we have to maintain our own
image and everything that entails, storing it to a docker register, version updates etc.