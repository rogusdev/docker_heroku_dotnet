# docker_heroku_dotnet
Demo how to build a docker dotnet core service deployed to heroku through circleci with postgres and redis attached

- use vagrant to start environment in which to run docker
- ^ docker is NOT a VM!  (it can and frequently does run IN a VM)
- use docker image to bootstrap new dotnet core app
- deploy to heroku
- deploy using circleci


1. Install VirtualBox: https://www.virtualbox.org/wiki/Downloads
1. Install Vagrant: https://www.vagrantup.com/downloads.html

```
git clone git@github.com:rogusdev/docker_heroku_dotnet.git
cd docker_heroku_dotnet/
cp ../Vagrantfile ./
cp ../README.md ./

vagrant up
chmod 600 .vagrant/machines/default/virtualbox/private_key
scp -r -i .vagrant/machines/default/virtualbox/private_key -P 2222 ~/.ssh/id_rsa* ubuntu@127.0.0.1:/home/ubuntu/.ssh/
vagrant ssh


cat << EOF > .env
PORT=5000
DATABASE_URL=Host=localhost;Username=postgres;Password=;Database=postgres
EOF

cat << EOF > .dockerignore
.DS_Store
.dockerignore
.env
.git
.gitignore
.idea
.vagrant
.vs
.vscode
docker-compose.yml
**/bin/
**/obj/
EOF

wget https://raw.githubusercontent.com/github/gitignore/master/VisualStudio.gitignore -O .gitignore

cat << EOF >> .gitignore
.DS_Store
.env
.vagrant
.vscode
**/bin/
**/obj/
EOF

cat << EOF > .gitattributes
# https://stackoverflow.com/questions/170961/whats-the-best-crlf-carriage-return-line-feed-handling-strategy-with-git
*        text eol=lf

*.cs     text diff=csharp
*.html   text diff=html
*.css    text diff=css
*.js     text diff=js
*.sql    text diff=sql

*.csproj text merge=union
*.sln    text merge=union eol=crlf
EOF

cat << EOF > app.json
{
  "name": "Docker Dotnet Core Heroku",
  "description": "An example app.json for container-deploy",
  "image": "microsoft/aspnetcore:2.0.3",
  "addons": [
    "heroku-postgresql"
  ]
}
EOF


cat << EOF > Dockerfile
# https://docs.docker.com/engine/examples/dotnetcore/#create-a-dockerfile-for-an-aspnet-core-application
# https://hub.docker.com/r/microsoft/dotnet/
# https://hub.docker.com/r/microsoft/aspnetcore-build/
# https://docs.microsoft.com/en-us/dotnet/core/docker/building-net-docker-images
# https://docs.microsoft.com/en-us/dotnet/core/deploying/index
# https://github.com/dotnet/dotnet-docker-samples/tree/master/aspnetapp
FROM microsoft/aspnetcore-build:2.0.3 AS build-env
WORKDIR /app

COPY *.sln .
COPY App.Web/*.csproj App.Web/
COPY App.Tests/*.csproj App.Tests/
RUN dotnet restore

COPY . .
RUN dotnet publish -c Release -o out \
    && touch App.Web/out/.env

FROM microsoft/aspnetcore:2.0.3
WORKDIR /app
COPY --from=build-env /app/App.Web/out .

# https://www.ctl.io/developers/blog/post/dockerfile-entrypoint-vs-cmd/
# Heroku will not accept an empty array CMD! -- must pass empty string inside
# but it actually just straight up ignores ENTRYPOINT anyway!  grr
CMD ["dotnet", "App.Web.dll"]
#CMD [""]
EOF


cat << EOF > docker-compose.yml
version: '3'
services:
  postgres:
    image: postgres:9.6
    ports:
     - "5432:5432"
  redis:
    image: redis:4.0.2
    ports:
     - "6379:6379"
  web:
    build: .
    ports:
      - "5000:5000"
    env_file: ".env"
    depends_on:
      - postgres
      - redis
EOF


docker run --rm -it -v /vagrant:/app microsoft/aspnetcore-build:2.0.3 sh -c 'cd /app; bash'


dotnet new sln -n App

dotnet new web -f netcoreapp2.0 -n App.Web
dotnet sln add App.Web/App.Web.csproj

dotnet new xunit -f netcoreapp2.0 -n App.Tests
dotnet add ./App.Tests package xunit --version 2.3.1  # update the version
dotnet add ./App.Tests package FakeItEasy --version 4.2.0
dotnet add ./App.Tests reference ./App.Web/App.Web.csproj
dotnet sln add App.Tests/App.Tests.csproj

dotnet add ./App.Tests package Microsoft.NET.Test.Sdk --version 15.5.0
dotnet add ./App.Tests package xunit.runner.visualstudio --version 2.3.1 

#sed -i '/PackageReference Include="Microsoft.NET.Test.Sdk"/ d' App.Tests/App.Tests.csproj
#sed -i 's/<PackageReference Include="xunit.runner.visualstudio" Version="2.3.1" \/>/<DotNetCliToolReference Include="dotnet-xunit" Version="2.3.1" \/>/' App.Tests/App.Tests.csproj

dotnet add ./App.Web package DotNetEnv --version 1.1.0
dotnet add ./App.Web package Npgsql --version 3.2.5
dotnet add ./App.Web package Dapper --version 1.50.2
dotnet add ./App.Web package StackExchange.Redis --version 1.2.6
dotnet add ./App.Web package Confluent.Kafka --version 0.11.3
dotnet add ./App.Web package Newtonsoft.Json --version 10.0.3

dotnet remove ./App.Web package Microsoft.AspNetCore.All
dotnet add ./App.Web package Microsoft.AspNetCore.Hosting --version 2.0.0
dotnet add ./App.Web package Microsoft.AspNetCore.Owin --version 2.0.0
dotnet add ./App.Web package Microsoft.AspNetCore.Server.Kestrel --version 2.0.0
dotnet add ./App.Web package Microsoft.Extensions.CommandLineUtils --version 1.1.1
dotnet add ./App.Web package Nancy --version 2.0.0-clienteastwood


cat << EOF > App.Web/Startup.cs
using DotNetEnv;
using Microsoft.AspNetCore.Builder;
using Nancy.Owin;

namespace App.Web
{
    public class Startup
    {
        public void Configure(IApplicationBuilder app)
        {
            Env.Load();
            app.UseOwin(x => x.UseNancy());
        }
    }
}
EOF

cat << EOF > App.Web/Program.cs
using System.IO;
using Microsoft.AspNetCore.Hosting;

namespace App.Web
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var port = System.Environment.GetEnvironmentVariable("PORT") ?? "5000";
            var host = new WebHostBuilder()
                .UseKestrel()
                .UseUrls($"http://*:{port}")
                .UseContentRoot(Directory.GetCurrentDirectory())
                .UseStartup<Startup>()
                .Build();

            host.Run();
        }
    }
}
EOF

cat << EOF > App.Web/BaseModule.cs
using System;
using Nancy;

namespace App.Web
{
    public class BaseModule : NancyModule
    {
        public BaseModule() : base("/")
        {
            Get("/", args => "NancyFX Hello @ " + DateTime.UtcNow);
        }
    }
}
EOF

cat << EOF > App.Web/Bootstrapper.cs
using Nancy;
using Nancy.Bootstrapper;
using Nancy.Configuration;
using Nancy.Diagnostics;
using Nancy.Json;
using Nancy.TinyIoc;

namespace App.Web
{
    public class Bootstrapper : DefaultNancyBootstrapper
    {
        protected override void ApplicationStartup(TinyIoCContainer container, IPipelines pipelines)
        {
        }

        public override void Configure(INancyEnvironment environment)
        {
            environment.Json(retainCasing: true);
            environment.Diagnostics(true, "password");
            environment.Tracing(enabled: true, displayErrorTraces: true);
            base.Configure(environment);
        }
    }
}
EOF


docker-compose build
docker-compose up


docker ps -a && docker images
#docker stop postgres:9.6
#docker rm postgres:9.6
#docker rmi ...

docker build -t docker-heroku-dotnet .

docker run --rm -it --env-file .env -p 0.0.0.0:5000:5000 docker-heroku-dotnet


heroku plugins:install heroku-container-registry
heroku login
heroku container:login

heroku create rogusdev-docker-heroku-dotnet
#heroku apps:destroy rogusdev-docker-heroku-dotnet
# https://rogusdev-docker-heroku-dotnet.herokuapp.com/

# Look at the "stack" in "Info" on Heroku's settings for this app, under config vars:
#  https://dashboard.heroku.com/apps/rogusdev-docker-heroku-dotnet/settings

heroku container:push web
```


If you are not in a vagrant vm, you might need to remove postgres:
```
# https://askubuntu.com/questions/32730/how-to-remove-postgres-from-my-installation
dpkg -l | grep postgres

sudo apt-get --purge remove postgresql postgresql-common postgresql-client-common
```

Integrating with CircleCI is possible but actually quite complicated -- these are just stubs I will fill in soon, the links tell the story for now:
```
# https://circleci.com/docs/2.0/deployment_integrations/#heroku
cat << EOF1 > .circleci/setup-heroku.sh
#!/bin/bash
wget https://cli-assets.heroku.com/branches/stable/heroku-linux-amd64.tar.gz
sudo mkdir -p /usr/local/lib /usr/local/bin
sudo tar -xvzf heroku-linux-amd64.tar.gz -C /usr/local/lib
sudo ln -s /usr/local/lib/heroku/bin/heroku /usr/local/bin/heroku

cat > ~/.netrc << EOF
machine api.heroku.com
  login $HEROKU_LOGIN
  password $HEROKU_API_KEY
EOF

cat >> ~/.ssh/config << EOF
VerifyHostKeyDNS yes
StrictHostKeyChecking no
EOF
EOF1

# https://circleci.com/docs/2.0/executor-types/#using-docker
cat << EOF > .circleci/config.yml
version: 2
jobs:
  build:
    machine: true
    steps:
      - checkout
      - run:
          name: Greeting
          command: echo Hello, world.
      - run:
          name: Print the Current Time
          command: date
EOF
```
