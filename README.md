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
DATABASE_URL=postgres://postgres:@postgres:5432
REDIS_URL=redis://:@redis:6379
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


# inside the new shell:

dotnet new sln -n App

dotnet new web -f netcoreapp2.0 -n App.Web
dotnet sln add App.Web/App.Web.csproj

dotnet new xunit -f netcoreapp2.0 -n App.Tests
dotnet add ./App.Tests package xunit --version 2.3.1  # update the version
dotnet add ./App.Tests package FakeItEasy --version 4.2.0
dotnet add ./App.Tests package Nancy.Testing --version 2.0.0-clienteastwood
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

# exit from docker bash


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


vagrant halt
vagrant destroy
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


Actually leverage the postgres and redis connections:
```
cat << EOF > App.Web/PostgresModule.cs
using System;
using Dapper;
using Nancy;

namespace App.Web
{
    public class PostgresModule : NancyModule
    {
        public PostgresModule() : base("/postgres")
        {
            // this is very bad REST, I know
            Get("/{key}/{value}", args =>
            {
                var key = args.key.ToString();
                var value = args.value.ToString();

                var databaseUrlString = Environment.GetEnvironmentVariable("DATABASE_URL");
                Console.WriteLine($"DATABASE_URL from ENV: {databaseUrlString}");
                // https://devcenter.heroku.com/articles/connecting-to-relational-databases-on-heroku-with-java#using-the-database_url-in-plain-jdbc
                var dbUri = new Uri(databaseUrlString);
                var userInfo = dbUri.UserInfo.Split(':');
                var npgsqlConnString = $"Host={dbUri.Host};Port={dbUri.Port};Database={dbUri.LocalPath.Substring(1)};Username={userInfo[0]};Password={userInfo[1]}";
                Console.WriteLine($"constructed npgsqlConnString: {npgsqlConnString}");

                // https://github.com/StackExchange/Dapper
                // http://dapper-tutorial.net/dapper
                using (var dbConn = new Npgsql.NpgsqlConnection(npgsqlConnString))
                {
                    dbConn.Open();
                    // # https://stackoverflow.com/questions/8902674/manually-map-column-names-with-class-properties/34536863#34536863
                    Dapper.DefaultTypeMap.MatchNamesWithUnderscores = true;

                    var now = DateTime.UtcNow;
                    var id = Guid.NewGuid();

                    // Insert some data
                    var newThing = new Thing()
                    {
                        Id = id,
                        Name = $"Hello world {now}: {key} = {value}",
                        Enabled = true,
                        CreatedAt = now.AddMinutes(-10),
                        UpdatedAt = now.AddMinutes(10),
                    };
                    dbConn.Execute(
                        "INSERT INTO things" +
                        " (id, name, enabled, created_at, updated_at)" +
                        " VALUES (@Id, @Name, @Enabled, @CreatedAt, @UpdatedAt)",
                        newThing
                    );
                    Console.WriteLine("inserted via dapper: {0}: {1}={2}", now, key, value);

                    // Retrieve all rows
                    var things = dbConn.Query<Thing>("SELECT * FROM things");
                    foreach (var thing in things)
                        Console.WriteLine(thing);
                    Console.WriteLine("read all dapper");

                    var addedThing = dbConn.QuerySingleOrDefault<Thing>(
                        "SELECT * FROM things WHERE id = @Id",
                        new { Id = id }
                    );
                    Console.WriteLine("added: {0}", addedThing);
                }

                return DateTime.UtcNow.ToString();
            });
        }

        private class Thing
        {
            public Guid Id { get; set; }
            public string Name { get; set; }
            public bool Enabled { get; set; }
            public DateTime CreatedAt { get; set; }
            public DateTime UpdatedAt { get; set; }

            public override string ToString()
            {
                return $"{nameof(Id)}: {Id}, {nameof(Name)}: {Name}, {nameof(Enabled)}: {Enabled}, {nameof(CreatedAt)}: {CreatedAt}, {nameof(UpdatedAt)}: {UpdatedAt}";
            }
        }
    }
}
EOF


cat << EOF > App.Web/RedisModule.cs
using System;
using System.Collections.Generic;
using Nancy;

namespace App.Web
{
    public class RedisModule : NancyModule
    {
        public RedisModule() : base("/redis")
        {
            // this is very bad REST, I know
            Get("/{key}/{value}", args =>
            {
                var key = args.key.ToString();
                var value = args.value.ToString();

                // https://github.com/redis/redis-rb/blob/master/lib/redis/client.rb#L408
                // https://msdn.microsoft.com/en-us/library/system.uri(v=vs.110).aspx
                // https://stackexchange.github.io/StackExchange.Redis/Configuration
                var redisUrlString = Environment.GetEnvironmentVariable("REDIS_URL");
                Console.WriteLine($"REDIS_URL from ENV: {redisUrlString}");
                var redisUri = new Uri(redisUrlString);
                var userInfo = redisUri.UserInfo.Split(':');
                var redisConnString = $"{redisUri.Host}:{redisUri.Port},password={userInfo[1]}";
                Console.WriteLine($"constructed redisConnString: {redisConnString}");

                var redisConn = StackExchange.Redis.ConnectionMultiplexer.Connect(redisConnString);
                var redisDb = redisConn.GetDatabase();
                redisDb.StringSet(key, value);
                var dict = new Dictionary<string, string>
                {
                    {args.key, redisDb.StringGet(key)},
                    {"ticks", DateTime.UtcNow.Ticks.ToString()},
                };

                // https://github.com/NancyFx/Nancy/wiki/Content-Negotiation
                var response = Response.AsJson(dict);
                response.ContentType = "application/json";
                return response;
            });
        }
    }
}
EOF


# poor man's redis-cli
echo "KEYS *" | nc -v localhost 6379
echo "GET key1" | nc -v localhost 6379


sudo apt-get install -y postgresql-client #redis-tools

cat << EOF | psql -h localhost -p 5432 -U postgres
CREATE TABLE things (
 id UUID PRIMARY KEY,
 name VARCHAR (255) NOT NULL,
 enabled BOOLEAN NOT NULL DEFAULT 'f',
 created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
 updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
EOF

echo "SELECT * FROM things;" | psql -h localhost -p 5432 -U postgres


heroku plugins:install heroku-redis

# https://elements.heroku.com/addons/heroku-redis
# https://devcenter.heroku.com/articles/heroku-redis#provisioning-the-add-on
# must add billing info before you can provision this
heroku addons:create heroku-redis:hobby-dev -a rogusdev-docker-heroku-dotnet

heroku redis:credentials -a rogusdev-docker-heroku-dotnet

# https://elements.heroku.com/addons/heroku-postgresql
# https://devcenter.heroku.com/articles/heroku-postgresql#version-support-and-legacy-infrastructure
heroku addons:create heroku-postgresql:hobby-dev -a rogusdev-docker-heroku-dotnet
# --version 9.6
# ^ specifying version gasve me 9.6.1, without specifying it gave me 9.6.4
# -- trying to specify minor version threw an error about not available

heroku pg:psql -a rogusdev-docker-heroku-dotnet

```


And now with tests!
```
rm App.Tests/UnitTest1.cs

cat << EOF > App.Tests/BaseModuleTests.cs
using System;
using App.Web;
using Nancy;
using Nancy.Testing;
using Xunit;

namespace App.Tests
{
    public class BaseModuleTests
    {
        private readonly Browser _browser;
        private readonly DateTime _utcNow;
        
        public BaseModuleTests()
        {
            _utcNow = DateTime.UtcNow;
            var timeProvider = new FixedTimeProvider(_utcNow);

            _browser = new Browser(with =>
            {
                with.Module<BaseModule>();
                with.Dependency(timeProvider);
            });
        }

        [Fact]
        public void ShouldAnnounceHello()
        {
            var response = _browser.Get("/", with => {
                with.HttpRequest();
            }).Result;
        
            Assert.Equal(HttpStatusCode.OK, response.StatusCode);
            // https://stackoverflow.com/questions/38443123/c-sharp-getting-http-body-using-nancy
            //Assert.StartsWith("NancyFX Hello @ ", response.Body.AsString());
            Assert.Equal($"NancyFX Hello @ {_utcNow}", response.Body.AsString());
        }
    }
}
EOF

cat << EOF > App.Tests/PostgresModuleTests.cs
using System;
using App.Web;
using FakeItEasy;
using Nancy;
using Nancy.Testing;
using Xunit;

namespace App.Tests
{
    public class PostgresModuleTests
    {
        private readonly Browser _browser;
        private readonly DateTime _utcNow;
        private readonly IPostgresDb _postgresDb;

        public PostgresModuleTests()
        {
            _postgresDb = A.Fake<IPostgresDb>();
            _utcNow = DateTime.UtcNow;
            var timeProvider = new FixedTimeProvider(_utcNow);
            
            // http://www.marcusoft.net/2013/01/NancyTesting2.html
            _browser = new Browser(with =>
            {
                with.Module<PostgresModule>();
                with.Dependency(timeProvider);
                with.Dependency(_postgresDb);
            });
        }

        [Fact]
        public void ShouldStoreAndRetrieveFromPostgres()
        {
            // https://fakeiteasy.readthedocs.io/en/stable/invoking-custom-code/
            // https://stackoverflow.com/questions/43781237/why-cant-i-capture-a-fakeiteasy-expectation-in-a-variable/43781563#43781563
            // http://thorarin.net/blog/post/2014/09/18/capturing-method-arguments-on-your-fakes-using-fakeiteasy.aspx
            Thing newThing = null;
            A.CallTo(() => _postgresDb.InsertThing(A<Thing>._))
                .Invokes(call => newThing = call.GetArgument<Thing>(0));
            A.CallTo(() => _postgresDb.AllThings()).ReturnsLazily(() => new[] { newThing });
            var id = Guid.Empty;
            A.CallTo(() => _postgresDb.FindThing(A<Guid>._))
                .Invokes(call => id = call.GetArgument<Guid>(0))
                .ReturnsLazily(() => newThing);

            var response = _browser.Get("/postgres/key1/value1", with => {
                with.HttpRequest();
            }).Result;

            Assert.Equal(HttpStatusCode.OK, response.StatusCode);
            // https://github.com/xunit/xunit/blob/master/test/test.xunit.assert/Asserts/StringAssertsTests.cs
            Assert.Matches(@"^\d{2}/\d{2}/\d{4} \d{2}:\d{2}:\d{2}$", response.Body.AsString());
            
            Assert.NotEqual(Guid.Empty, id);
            Assert.Equal(newThing.Id, id);
            Assert.Equal($"Hello world {_utcNow}: key1 = value1", newThing.Name);
            Assert.True(newThing.Enabled);
            Assert.Equal(_utcNow.AddMinutes(-10), newThing.CreatedAt);
            Assert.Equal(_utcNow.AddMinutes(10), newThing.UpdatedAt);
        }
    }
}
EOF

cat << EOF > App.Tests/RedisModuleTests.cs
using System;
using System.Collections.Generic;
using App.Web;
using FakeItEasy;
using Nancy;
using Nancy.Json;
using Nancy.Testing;
using Newtonsoft.Json;
using Xunit;

namespace App.Tests
{
    public class RedisModuleTests
    {
        private readonly Browser _browser;
        private readonly DateTime _utcNow;
        private readonly IRedisDb _redisDb;

        public RedisModuleTests()
        {
            _redisDb = A.Fake<IRedisDb>();
            _utcNow = DateTime.UtcNow;
            var timeProvider = new FixedTimeProvider(_utcNow);

            _browser = new Browser(with =>
            {
                with.Module<RedisModule>();
                with.Dependency(timeProvider);
                with.Dependency(_redisDb);
                // https://github.com/NancyFx/Nancy/pull/1489
                //with.ApplicationStartup((x, y) => JsonSettings.RetainCasing = true);
                //with.Configure(env => env.Json(retainCasing: true));
            });
        }

        [Fact]
        public void ShouldStoreAndRetrieveFromRedis()
        {
            A.CallTo(() => _redisDb.StringGet("key1")).Returns("value1");
            
            var response = _browser.Get("/redis/key1/value1", with => {
                with.HttpRequest();
            }).Result;

            // https://stackoverflow.com/questions/1207731/how-can-i-deserialize-json-to-a-simple-dictionarystring-string-in-asp-net
            // https://stackoverflow.com/questions/31089347/testing-a-rest-api-in-nancy
            //var responseDict = response.Body.DeserializeJson<Dictionary<string, string>>();  // empty dict for me
            var responseDict = JsonConvert.DeserializeObject<Dictionary<string, string>>(response.Body.AsString());
            var expectedDict = new Dictionary<string, string>
            {
                {"key1", "value1"},
                {"ticks", _utcNow.Ticks.ToString()},
            };

            A.CallTo(() => _redisDb.StringSet("key1", "value1")).MustHaveHappened();

            Assert.Equal(HttpStatusCode.OK, response.StatusCode);
            // https://github.com/xunit/xunit/blob/master/test/test.xunit.assert/Asserts/CollectionAssertsTests.cs
            //Assert.True(response.Headers.ContainsKey("content-type"));
            //Assert.Equal("application/json", response.Headers["content-type"]);
            Assert.Equal(expectedDict, responseDict);
        }
    }
}
EOF


# But we do need to update the implementations to make them more easily testable:

cat << EOF > App.Web/IProvideTime.cs
using System;

namespace App.Web
{
    public interface IProvideTime
    {
        DateTime UtcNow { get; }
    }

    public class SystemTimeProvider : IProvideTime
    {
        public DateTime UtcNow => DateTime.UtcNow;
    }

    public class FixedTimeProvider : IProvideTime
    {
        public DateTime UtcNow { get; private set; }

        public FixedTimeProvider(DateTime dateTime)
        {
            UtcNow = dateTime;
        }

        public void Update(DateTime dateTime)
        {
            UtcNow = dateTime;
        }

        public void Add(TimeSpan timeSpan)
        {
            UtcNow += timeSpan;
        }
    }
}
EOF

cat << EOF > App.Web/BaseModule.cs
using Nancy;

namespace App.Web
{
    public class BaseModule : NancyModule
    {
        public BaseModule(IProvideTime timeProvider) : base("/")
        {
            Get("/", args => "NancyFX Hello @ " + timeProvider.UtcNow);
        }
    }
}
EOF

cat << EOF > App.Web/PostgresModule.cs
using System;
using System.Collections.Generic;
using System.Data;
using Dapper;
using Nancy;

namespace App.Web
{
    public class PostgresModule : NancyModule
    {
        public PostgresModule(IProvideTime timeProvider, IPostgresDb postgresDb) : base("/postgres")
        {
            // this is very bad REST, I know
            Get("/{key}/{value}", args =>
            {
                var key = args.key.ToString();
                var value = args.value.ToString();

                var now = timeProvider.UtcNow;
                var id = Guid.NewGuid();

                // Insert some data
                var newThing = new Thing
                {
                    Id = id,
                    Name = $"Hello world {now}: {key} = {value}",
                    Enabled = true,
                    CreatedAt = now.AddMinutes(-10),
                    UpdatedAt = now.AddMinutes(10),
                };

                postgresDb.InsertThing(newThing);
                Console.WriteLine("inserted via dapper: {0}: {1}={2}", now, key, value);

                var things = postgresDb.AllThings();
                foreach (var thing in things)
                    Console.WriteLine(thing);
                Console.WriteLine("read all dapper");

                var addedThing = postgresDb.FindThing(id);
                Console.WriteLine("added: {0}", addedThing);

                return timeProvider.UtcNow.ToString();
            });
        }
    }

    public class Thing
    {
        public Guid Id { get; set; }
        public string Name { get; set; }
        public bool Enabled { get; set; }
        public DateTime CreatedAt { get; set; }
        public DateTime UpdatedAt { get; set; }

        public override string ToString()
        {
            return $"{nameof(Id)}: {Id}, {nameof(Name)}: {Name}, {nameof(Enabled)}: {Enabled}, {nameof(CreatedAt)}: {CreatedAt}, {nameof(UpdatedAt)}: {UpdatedAt}";
        }
    }

    public interface IPostgresDb
    {
        void InsertThing(Thing thing);
        IEnumerable<Thing> AllThings();
        Thing FindThing(Guid id);
    }

    public class PostgresDb : IPostgresDb
    {
        private readonly string _npgsqlConnString;

        public PostgresDb()
        {
            var databaseUrlString = Environment.GetEnvironmentVariable("DATABASE_URL");
            Console.WriteLine($"DATABASE_URL from ENV: {databaseUrlString}");
            // https://devcenter.heroku.com/articles/connecting-to-relational-databases-on-heroku-with-java#using-the-database_url-in-plain-jdbc
            var dbUri = new Uri(databaseUrlString);
            var userInfo = dbUri.UserInfo.Split(':');
            _npgsqlConnString = $"Host={dbUri.Host};Port={dbUri.Port};Database={dbUri.LocalPath.Substring(1)};Username={userInfo[0]};Password={userInfo[1]}";
            Console.WriteLine($"constructed npgsqlConnString: {_npgsqlConnString}");

            // # https://stackoverflow.com/questions/8902674/manually-map-column-names-with-class-properties/34536863#34536863
            DefaultTypeMap.MatchNamesWithUnderscores = true;
        }

        private IDbConnection NewConn()
        {
            // http://www.npgsql.org/doc/connection-string-parameters.html#pooling
            // http://www.npgsql.org/doc/performance.html
            return new Npgsql.NpgsqlConnection(_npgsqlConnString);
        }

        public void InsertThing(Thing newThing)
        {
            // https://github.com/StackExchange/Dapper
            // http://dapper-tutorial.net/dapper
            using (var dbConn = NewConn())
            {
                dbConn.Open();

                dbConn.Execute(
                    "INSERT INTO things" +
                    " (id, name, enabled, created_at, updated_at)" +
                    " VALUES (@Id, @Name, @Enabled, @CreatedAt, @UpdatedAt)",
                    newThing
                );
            }
        }

        public IEnumerable<Thing> AllThings()
        {
            using (var dbConn = NewConn())
            {
                dbConn.Open();

                return dbConn.Query<Thing>("SELECT * FROM things");
            }
        }
        
        public Thing FindThing(Guid id)
        {
            using (var dbConn = NewConn())
            {
                dbConn.Open();

                return dbConn.QuerySingleOrDefault<Thing>(
                    "SELECT * FROM things WHERE id = @Id",
                    new { Id = id }
                );
            }
        }
    }
}
EOF

cat << EOF > App.Web/RedisModule.cs
using System;
using System.Collections.Generic;
using Nancy;
using StackExchange.Redis;

namespace App.Web
{
    public class RedisModule : NancyModule
    {
        public RedisModule(IProvideTime timeProvider, IRedisDb redisDb) : base("/redis")
        {
            // this is very bad REST, I know
            Get("/{key}/{value}", args =>
            {
                var key = args.key.ToString();
                var value = args.value.ToString();

                redisDb.StringSet(key, value);
                Console.WriteLine("redis set: {0}={1}", key, value);

                var redisValue = redisDb.StringGet(key);
                Console.WriteLine("redis get: {0}", redisValue);

                var dict = new Dictionary<string, string>
                {
                    {key, redisValue},
                    {"ticks", timeProvider.UtcNow.Ticks.ToString()},
                };

                // https://github.com/NancyFx/Nancy/wiki/Content-Negotiation
                var response = Response.AsJson(dict);
                //response.ContentType = "application/json";
                return response;
            });
        }
    }

    public interface IRedisDb
    {
        void StringSet(string key, string value);
        string StringGet(string key);
    }

    public class RedisDb : IRedisDb
    {
        private readonly Lazy<IDatabase> _redisDb;

        public RedisDb()
        {
            // https://github.com/redis/redis-rb/blob/master/lib/redis/client.rb#L408
            // https://msdn.microsoft.com/en-us/library/system.uri(v=vs.110).aspx
            // https://stackexchange.github.io/StackExchange.Redis/Configuration
            var redisUrlString = Environment.GetEnvironmentVariable("REDIS_URL");
            Console.WriteLine($"REDIS_URL from ENV: {redisUrlString}");
            var redisUri = new Uri(redisUrlString);
            var userInfo = redisUri.UserInfo.Split(':');
            var redisConnString = $"{redisUri.Host}:{redisUri.Port},password={userInfo[1]}";
            Console.WriteLine($"constructed redisConnString: {redisConnString}");

            _redisDb = new Lazy<IDatabase>(() =>
                ConnectionMultiplexer.Connect(redisConnString).GetDatabase());
        }

        public void StringSet(string key, string value)
        {
            _redisDb.Value.StringSet(key, value);
        }

        public string StringGet(string key)
        {
            return _redisDb.Value.StringGet(key);
        }
    }
}
EOF

dotnet clean && dotnet test
```
