# Lab 6: Build automation and Docker image builds

In this exercises you will:

1. build a **.NET** and a **Java** application by hand with their build automation tools, to understand which steps a build consists of;
2. write a **Dockerfile** that performs the same steps in a clean, reproducible environment;
3. **break down** what happens during `docker build`, layer by layer;
4. optimize the build: layer cache, `.dockerignore`, multi-stage builds, build arguments, test stages and BuildKit mounts;
5. tag an image and publish it to a container registry **manually**.

The commands you write here will be **automated in a CI pipeline (GitHub Actions) in the next exercise**. Everything in this lab is therefore done from your terminal. The better you understand each manual step, the easier it will be to automate it.

> **No application logic is written in this lab.** You will generate minimal sample applications from official templates. The goal is the build process, not the code.

> **Prerequisites:** the project repository from the earlier labs, git, a GitHub account, and on your computer: **Docker** (Docker Desktop on Windows/macOS, Docker Engine on Linux), the **.NET SDK 10**, and a **JDK 21** (or newer). Maven is not needed, because the project includes the Maven wrapper. If you cannot install the SDKs locally, you can do Tasks 1 to 7 inside a GitHub Codespace, or skip the local builds in Task 1 and do only the Docker parts, since the Docker builds contain their own SDKs.

---

## Concepts

### What is a build?

A build turns source code into something that can be run or delivered. Most builds consist of the same stages, whatever the technology:

```text
 source  →  restore/resolve  →  compile  →  (test)  →  package/publish  →  artifact / image
 code        dependencies                                                     (.dll, .jar, Docker image)
```

### Build automation tools

A **build automation tool** describes these stages in configuration files and executes them in a repeatable way from the command line.

| Tool | Ecosystem | Configuration file(s) | Typical commands |
|------|-----------|-----------------------|------------------|
| **dotnet CLI / MSBuild** | .NET | `*.csproj`, `*.sln`/`*.slnx`, `global.json`, `Directory.Build.props`, `nuget.config` | `dotnet restore`, `build`, `test`, `publish` |
| **Maven** | Java | `pom.xml`, wrapper (`mvnw`, `.mvn/`) | `mvn compile`, `test`, `package`, `verify` |
| **Gradle** | Java/Kotlin | `build.gradle(.kts)`, `settings.gradle(.kts)`, wrapper (`gradlew`) | `gradle build`, `test`, `assemble` |
| **npm** | JavaScript | `package.json`, `package-lock.json` | `npm ci`, `npm run build` |
| **Docker (BuildKit)** | any | `Dockerfile`, `.dockerignore` | `docker build`, `docker buildx build` |
| **Make** | any | `Makefile` | `make build` |

Two good habits that apply to all of them:

- use a **wrapper** or a pinned tool version (`mvnw`, `gradlew`, `global.json`), so that everyone uses the same version;
- **lock** the dependency versions (`packages.lock.json`, a Maven BOM/`<dependencyManagement>`, `package-lock.json`), so that builds are reproducible.

Docker sits one level above the others: a Dockerfile does not replace `dotnet` or `mvn`. It **runs them inside a controlled environment** and packages the result together with its runtime.

### Docker vocabulary

| Term | Meaning |
|------|---------|
| **Image** | A read-only template: a stack of **layers** plus metadata (default command, environment variables, exposed ports, user) |
| **Layer** | A set of filesystem changes created by one instruction. Layers are stored once and shared between images |
| **Container** | A running instance of an image |
| **Dockerfile** | A text file with the instructions for building an image |
| **Build context** | The folder whose content is sent to the builder. `COPY` can only use files from it |
| **Registry** | A server that stores images (Docker Hub, GitHub Container Registry `ghcr.io`) |
| **Tag** | A human-readable, **movable** name of an image version, e.g. `my-app:1.0.0` |
| **Digest** | The **immutable** SHA-256 identifier of an image, e.g. `my-app@sha256:ab12...` |
| **BuildKit** | The build engine behind `docker build` (the default in current Docker versions) |

Check your installation before continuing:

```bash
docker version
docker run --rm hello-world
```

---

## Task 0: Preparation

Create a new public repository `docker-build-demo` and clone it. Use it for Tasks 1 to 7. Its final structure will look like this:

```text
docker-build-demo/
├── dotnet-app/          # generated in Task 1
│   ├── src/DemoApi/
│   ├── tests/DemoApi.Tests/
│   ├── Dockerfile.naive
│   ├── Dockerfile.cache
│   ├── Dockerfile
│   └── .dockerignore
└── java-app/            # generated in Task 1
    ├── src/
    ├── pom.xml
    ├── Dockerfile.naive
    ├── Dockerfile
    └── .dockerignore
```

Work as in Lab 5: one branch per task, a pull request into `main` and a merge. Commit every Dockerfile you create, including the "bad" ones, because you will compare them.

---

## Task 1: The sample applications and their native builds

### 1.1 .NET

Generate a minimal web API with a test project:

```bash
mkdir dotnet-app && cd dotnet-app
dotnet new webapi  -n DemoApi       -o src/DemoApi
dotnet new xunit   -n DemoApi.Tests -o tests/DemoApi.Tests
dotnet add tests/DemoApi.Tests reference src/DemoApi/DemoApi.csproj
dotnet new sln -n DemoApp
dotnet sln add src/DemoApi tests/DemoApi.Tests
dotnet new gitignore
```

(On .NET 10 the solution file is created as `DemoApp.slnx`, on older SDKs as `DemoApp.sln`. Both work.)

The whole .NET build is four commands:

```bash
dotnet restore                                           # download NuGet packages
dotnet build   --configuration Release --no-restore      # compile
dotnet test    --configuration Release --no-build        # run the tests
dotnet publish src/DemoApi -c Release -o publish --no-build   # produce the deployable output
```

Run `dotnet publish/DemoApi.dll` (or `dotnet publish/DemoApi.dll` from inside the folder), open the address it prints and call `/weatherforecast`. Look at the `publish/` folder, and note the size of the folder. Add `bin/`, `obj/` and `publish/` to `.gitignore` if they are not already there.

Add a `Directory.Build.props` file in `dotnet-app/`. It applies one setting to every project in the folder tree, and it is the .NET way of centralizing build configuration:

```xml
<Project>
  <PropertyGroup>
    <Version>0.1.0</Version>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
</Project>
```

| File | Purpose |
|------|---------|
| `*.csproj` | Project definition: target framework (`net10.0`), package references, output type |
| `*.sln` / `*.slnx` | Groups the projects so that one command builds all of them |
| `global.json` | Pins the SDK version (create one with `dotnet new globaljson`) |
| `Directory.Build.props` | Settings shared by all projects below it |
| `nuget.config` | Package sources |

Run the build with `-c Debug` and `-c Release`. Note what differs (size of the DLL, `.pdb` files, optimizations).

### 1.2 Java

Generate a Spring Boot project. It is only a convenient and realistic source of a `pom.xml` and the Maven wrapper:

```bash
cd ..
curl https://start.spring.io/starter.zip \
  -d type=maven-project -d language=java -d javaVersion=21 \
  -d groupId=hr.demo -d artifactId=demo -d name=demo -d dependencies=web \
  -o demo.zip
unzip demo.zip -d java-app && rm demo.zip
```

(Alternatively, generate it on <https://start.spring.io> and unzip it into `java-app/`.)

```bash
cd java-app
./mvnw -B clean verify          # on Windows: mvnw.cmd -B clean verify
ls target/                      # demo-0.0.1-SNAPSHOT.jar and test reports
java -jar target/demo-0.0.1-SNAPSHOT.jar     # listens on port 8080; stop with Ctrl+C
```

Maven runs **phases** in a fixed order. Asking for a phase runs all the earlier ones first:

```text
validate → compile → test → package → verify → install → deploy
```

| Command | What runs |
|---------|-----------|
| `mvn compile` | validate, compile |
| `mvn test` | ... plus the unit tests |
| `mvn package` | ... plus creating the `.jar` in `target/` |
| `mvn verify` | ... plus integration checks |
| `mvn install` | ... plus copying the artifact into the local repository `~/.m2` |
| `mvn clean` | A separate lifecycle that deletes `target/` |

Open `pom.xml` and find: `<parent>` (where the dependency and plugin versions come from), `<properties>` (`java.version`), `<dependencies>` (what the project needs) and `<build><plugins>` (how it is built). Write down what each does for this project. Useful flags: `-B` (batch mode), `-ntp` (no transfer progress, shorter output), `-DskipTests`.

> **Gradle (optional comparison).** Gradle does the same with `build.gradle(.kts)` and `gradlew`: `./gradlew build` ≈ `./mvnw verify`, `./gradlew assemble` ≈ `./mvnw package -DskipTests`. Generate a Gradle variant with `type=gradle-project` in the command above and compare the two build files.

### Expected result of Task 1

- Both applications build and run on your machine.
- A list of the build steps for each ecosystem (restore, compile, test, package) with the exact command for each. **You will reuse these commands in the Dockerfiles.**
- A short note on the difference between `Debug` and `Release`.

---

## Task 2: Anatomy of a Dockerfile and of `docker build`

### 2.1 The instructions you need

| Instruction | What it does | Creates a layer with files? |
|-------------|--------------|-----------------------------|
| `FROM image:tag [AS name]` | Starts a new stage from a base image | the base image's layers |
| `WORKDIR /app` | Sets (and creates) the working directory for later instructions | metadata (plus the directory) |
| `COPY src dest` | Copies files from the build context (or `--from=stage`) | **yes** |
| `RUN command` | Runs a command at build time and keeps the resulting filesystem changes | **yes** |
| `ARG NAME=default` | A variable that exists only during the build (`--build-arg`) | no |
| `ENV NAME=value` | A variable that also exists in the running container | no (metadata) |
| `EXPOSE 8080` | Documents the port the application listens on (does **not** publish it) | no (metadata) |
| `USER name` | Sets the user for later instructions and for the container | no (metadata) |
| `ENTRYPOINT [...]` / `CMD [...]` | What the container runs at start (`ENTRYPOINT` is the program, `CMD` its default arguments) | no (metadata) |
| `HEALTHCHECK` | A command that Docker runs to check whether the container is healthy | no (metadata) |
| `LABEL k=v` | Metadata, e.g. source repository, version | no (metadata) |

Always use the **exec form** (`["dotnet", "DemoApi.dll"]`) for `ENTRYPOINT`/`CMD`. The shell form (`ENTRYPOINT dotnet DemoApi.dll`) runs the application under a shell, which, among other problems, does not forward the stop signal (`SIGTERM`) to it.

### 2.2 What happens during `docker build`? (the breakdown)

When you run `docker build -t demo .`, the following happens:

```text
 1. Send the BUILD CONTEXT (the folder, minus what .dockerignore excludes) to the builder
 2. Parse the Dockerfile into a build graph: one node per instruction, one branch per stage
 3. Resolve each FROM: contact the registry, find the digest for the tag, download missing layers
 4. For each instruction, compute a CACHE KEY
        = the instruction text + the parent layer + (for COPY/ADD) a checksum of the copied files
 5.   cache hit  → reuse the stored layer ("CACHED" in the output)
      cache miss → run the instruction in a temporary container, save the filesystem
                   difference as a NEW layer, and recompute every following step
 6. Independent stages run in parallel. Stages that the final stage does not use are skipped
 7. Assemble the final image: the list of layers (manifest) + configuration
        (ENTRYPOINT, ENV, USER, EXPOSE, ...), and give it the tag
 8. Load the image into the local image store (or push it to a registry)
```

Two rules follow from this. They are the key to fast builds:

> **Rule 1:** Once one layer is rebuilt (a cache miss), **every layer after it** is rebuilt too.
> **Rule 2:** So put the things that change **rarely** (dependency definitions) at the top, and the things that change **often** (source code) at the bottom.

Metadata instructions (`ENV`, `EXPOSE`, `USER`, `ENTRYPOINT`...) do not add files to the image. They are stored in the image configuration.

### 2.3 Reading the build output

Run any build with `--progress=plain` to see every step in full. Learn to read these parts: `[internal] load build definition` (the Dockerfile is read), `load build context` + `transferring context` (step 1 above), `[stage N/M]` (step per instruction), `CACHED` (cache hit), and `exporting to image` (step 7).

### Questions for your report

1. Which instructions in a Dockerfile produce layers that contain files, and which only change the image metadata? Why does that matter for the image size?
2. In your own words: why does Rule 1 make the *order* of instructions important?
3. What is the difference between `ENTRYPOINT` and `CMD`, and between `ENV` and `ARG`?

---

## Task 3: Step A: the naive Dockerfile

First write the simplest possible Dockerfile, so that you have a baseline to improve.

### 3.1 .NET

Create `dotnet-app/Dockerfile.naive`:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0
WORKDIR /app
COPY . .
RUN dotnet publish src/DemoApi/DemoApi.csproj -c Release -o out
ENV ASPNETCORE_HTTP_PORTS=8080
ENTRYPOINT ["dotnet", "out/DemoApi.dll"]
```

Build and run it:

```bash
cd dotnet-app
docker build -f Dockerfile.naive -t demo-dotnet:naive .
docker run --rm -p 8080:8080 demo-dotnet:naive
# in another terminal:
curl http://localhost:8080/weatherforecast        # an endpoint from the webapi template
```

### 3.2 Java

Create `java-app/Dockerfile.naive`:

```dockerfile
FROM maven:3-eclipse-temurin-21
WORKDIR /app
COPY . .
RUN mvn -B -ntp package -DskipTests
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "target/demo-0.0.1-SNAPSHOT.jar"]
```

```bash
cd java-app
docker build -f Dockerfile.naive -t demo-java:naive .
docker run --rm -p 8080:8080 demo-java:naive
```

### 3.3 Inspect the images

```bash
docker images                                       # SIZE of each image
docker history demo-dotnet:naive                    # one row per layer, with the size of each
docker image inspect demo-dotnet:naive --format '{{json .RootFS.Layers}}'
docker image inspect demo-dotnet:naive --format '{{json .Config.Entrypoint}} {{json .Config.User}}'
```

Record the sizes in your report and answer:

- Which layer is the largest? Why does a **runtime** image contain the whole SDK, the source code and the `obj/` folders?
- Which user does the application run as?
- List at least three problems with this image (size, security, speed of rebuilding).

### Expected result of Task 3

- Two working naive images with their `docker history` output saved in the report.
- The list of problems.

---

## Task 4: Step B: the layer cache and `.dockerignore`

### 4.1 Observe the cache (and break it)

Run the naive build again:

```bash
docker build -f Dockerfile.naive -t demo-dotnet:naive .
```

Every step is `CACHED`. Now change **one character** in `src/DemoApi/Program.cs` (for example, a comment) and rebuild with plain progress:

```bash
docker build --progress=plain -f Dockerfile.naive -t demo-dotnet:naive .
```

Which steps were rebuilt? Was the NuGet restore (it is part of `dotnet publish`) repeated? Why? Because of `COPY . .` and Rule 1, any file change invalidates the layer in which the dependencies were also downloaded.

### 4.2 `.dockerignore`

The build context of the naive build contains `bin/`, `obj/`, `.git` and possibly local secrets. Look at the `transferring context` line in the output. Create `dotnet-app/.dockerignore`:

```text
**/bin/
**/obj/
**/TestResults/
publish/
.git/
.gitignore
.vs/
.vscode/
**/*.md
Dockerfile*
.dockerignore
.env
```

Rebuild and compare the size of the transferred context. Two reasons to keep this file: **speed** (less data is sent) and **correctness/security** (a local `.env` or a `bin/` folder cannot end up inside an image or invalidate the cache). Create `java-app/.dockerignore` as well:

```text
target/
.git/
.gitignore
.idea/
.vscode/
**/*.md
Dockerfile*
.dockerignore
.env
```

### 4.3 Order the instructions for the cache

Copy only the file that defines the dependencies first, restore, and only then copy the source. Create `dotnet-app/Dockerfile.cache`:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0
WORKDIR /src

# 1. dependency definition only → this layer changes rarely
COPY Directory.Build.props ./
COPY src/DemoApi/DemoApi.csproj src/DemoApi/
RUN dotnet restore src/DemoApi/DemoApi.csproj

# 2. source code → this layer changes often
COPY src/DemoApi/ src/DemoApi/
RUN dotnet publish src/DemoApi/DemoApi.csproj -c Release -o /app --no-restore

ENV ASPNETCORE_HTTP_PORTS=8080
WORKDIR /app
ENTRYPOINT ["dotnet", "DemoApi.dll"]
```

Build it (`-t demo-dotnet:cache`), change `Program.cs`, rebuild and fill in the table with your own observations (`CACHED` or `rebuilt`):

| Change you make | `COPY ...csproj` | `RUN dotnet restore` | `COPY src/DemoApi/` | `RUN dotnet publish` |
|-----------------|------------------|----------------------|---------------------|----------------------|
| Nothing (rebuild) | | | | |
| Edit `Program.cs` | | | | |
| Add a NuGet package to the `.csproj` | | | | |
| Change the base image tag (`10.0` → `10.0.0`) | | | | |

### Expected result of Task 4

- The filled-in cache table and a one-paragraph conclusion: which change is the most expensive and why.
- Both `.dockerignore` files committed, and a note on how the build context shrank.

---

## Task 5: Step C: multi-stage builds

The cached Dockerfile is faster, but the final image still carries the whole SDK. A **multi-stage build** uses several `FROM` instructions in one Dockerfile. Early stages build the application. The last stage starts from a small **runtime-only** image and copies in only the published output. Everything else is thrown away.

```text
 stage "build"  (SDK image, ~800 MB)                 stage "final"  (ASP.NET runtime image, ~200 MB)
 ┌──────────────────────────────────┐                ┌──────────────────────────────┐
 │ SDK + NuGet cache + source + obj │  COPY --from   │ runtime                      │
 │ dotnet publish → /app/publish ───┼───────────────►│ + /app (published files only)│
 └──────────────────────────────────┘                └──────────────────────────────┘
        thrown away (not in final image)                      this becomes the image
```

### 5.1 .NET

Create `dotnet-app/Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1

############ Stage 1: build ############
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

COPY Directory.Build.props ./
COPY src/DemoApi/DemoApi.csproj src/DemoApi/
RUN dotnet restore src/DemoApi/DemoApi.csproj

COPY src/DemoApi/ src/DemoApi/
RUN dotnet publish src/DemoApi/DemoApi.csproj \
        -c Release -o /app/publish --no-restore

############ Stage 2: runtime ############
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final
WORKDIR /app

COPY --from=build /app/publish .

EXPOSE 8080
USER $APP_UID
ENTRYPOINT ["dotnet", "DemoApi.dll"]
```

```bash
docker build -t demo-dotnet:multistage .
docker images demo-dotnet
docker run --rm -p 8080:8080 demo-dotnet:multistage
```

Notes on the new instructions:

- `AS build` names a stage. `COPY --from=build` takes files **from another stage**, not from your computer.
- `USER $APP_UID` runs the application as a **non-root** user. The official ASP.NET images (since .NET 8) provide a user named `app`, and the `APP_UID` variable holds its numeric id. A process in a container should not run as root.
- Since .NET 8 the official ASP.NET images listen on port **8080**, which is why 8080 is exposed instead of 80.
- Build only up to a certain stage with `docker build --target build -t demo-dotnet:build .` (useful for debugging the build stage).

### 5.2 Java

Create `java-app/Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1

############ Stage 1: build ############
FROM maven:3-eclipse-temurin-21 AS build
WORKDIR /build

# 1. dependency definition only
COPY pom.xml .
RUN mvn -B -ntp dependency:go-offline

# 2. source code
COPY src ./src
RUN mvn -B -ntp package -DskipTests

############ Stage 2: runtime ############
FROM eclipse-temurin:21-jre AS final
WORKDIR /app

RUN groupadd --system app && useradd --system --gid app --no-create-home app

COPY --from=build /build/target/*.jar app.jar

USER app
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
cd java-app
docker build -t demo-java:multistage .
docker run --rm -p 8080:8080 demo-java:multistage
```

### 5.3 Compare

Fill in the table with your own measurements (use `docker images`, and time the rebuild after editing a source file):

| Image | Size | Contains SDK/JDK/Maven? | Runs as root? | Rebuild time after editing a source file |
|-------|------|--------------------------|---------------|-------------------------------------------|
| `demo-dotnet:naive` | | | | |
| `demo-dotnet:cache` | | | | |
| `demo-dotnet:multistage` | | | | |
| `demo-java:naive` | | | | |
| `demo-java:multistage` | | | | |

Answer:

1. Why is the JDK needed in the build stage but a **JRE** is enough in the final stage?
2. Why are tests skipped inside the image build (`-DskipTests`)? You will make a better decision in Task 6.
3. Why is `COPY pom.xml .` followed by `dependency:go-offline` before `COPY src ./src`?
4. Which instructions of the final stage only change the **metadata**, and which add files?
5. What happens to the `build` stage when the build finishes? Is it in the final image? (Check with `docker history`.)

> **Optional: a smaller and safer runtime.** The official **chiseled** .NET images are even smaller: no shell, no package manager, and non-root by default. Try a tag with the `-noble-chiseled` suffix for the `aspnet` image (check the exact tag in the Microsoft Artifact Registry or the Docker docs). Why can you no longer run `docker exec -it <container> sh` in such an image, and is that a feature or a limitation?

### Expected result of Task 5

- Both multi-stage Dockerfiles committed, and running containers.
- The comparison table and the answers.
- **An annotated Dockerfile:** copy one multi-stage Dockerfile into your report and add a comment to **every line**: what it does, whether it creates a layer with files, and whether it is cached when only the source code changes.

---

## Task 6: Step D: build parameters, test stages and BuildKit features

### 6.1 Build arguments

`ARG` makes a Dockerfile configurable without editing it. Note the **scope** rule: an `ARG` declared **before** the first `FROM` can be used only in `FROM` lines. Inside a stage it must be declared again.

Change `dotnet-app/Dockerfile` as follows (only the changed lines are shown):

```dockerfile
ARG DOTNET_VERSION=10.0

FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS build
ARG APP_VERSION=0.0.0
...
RUN dotnet publish src/DemoApi/DemoApi.csproj \
        -c Release -o /app/publish --no-restore -p:Version=${APP_VERSION}

FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION} AS final
ARG APP_VERSION=0.0.0
LABEL org.opencontainers.image.version=${APP_VERSION}
...
```

```bash
docker build --build-arg APP_VERSION=0.1.0 -t demo-dotnet:0.1.0 .
docker image inspect demo-dotnet:0.1.0 --format '{{json .Config.Labels}}'
```

Why must `APP_VERSION` be declared in both stages? What happens if you forget it in the `final` stage?

### 6.2 Labels

Labels are metadata for people and tools. The `org.opencontainers.image.*` names are standard. Add them to the final stage and **inspect** them afterwards:

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/<owner>/<repo>" \
      org.opencontainers.image.description="Demo API" \
      org.opencontainers.image.licenses="MIT"
```

When you later push an image to GitHub Container Registry, the `source` label is what links the package to your repository.

### 6.3 A test stage

Instead of skipping tests, a Dockerfile can contain a **separate stage that runs them**. Add it to `dotnet-app/Dockerfile`, between `build` and `final`:

```dockerfile
############ Test stage (optional) ############
FROM build AS test
COPY tests/DemoApi.Tests/ tests/DemoApi.Tests/
RUN dotnet test tests/DemoApi.Tests/DemoApi.Tests.csproj -c Release
```

and for Java (`java-app/Dockerfile`):

```dockerfile
FROM build AS test
RUN mvn -B -ntp test
```

Run the stages separately:

```bash
docker build --target test  -t demo-dotnet:test .       # runs the tests; fails if a test fails
docker build                -t demo-dotnet:multistage . # builds the final image, skips the test stage
```

Observe: BuildKit **skips stages that the final stage does not depend on**, so the tests run only when you target them. Make a test fail on purpose (`Assert.True(false);`) and confirm that `--target test` fails with a non-zero exit code (`echo $?` / `echo %ERRORLEVEL%`). A pipeline can use that exit code in the next exercise.

Answer: what is the advantage of running tests *inside* the Docker build instead of on your machine? What is the downside (hint: test reports, speed)?

### 6.4 BuildKit cache mounts

`dependency:go-offline` and `dotnet restore` cache dependencies **in an image layer**, so a change of `pom.xml` or the `.csproj` re-downloads everything. A **cache mount** keeps the download cache outside the layers, in a persistent cache directory that BuildKit manages:

```dockerfile
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B -ntp package -DskipTests
```

```dockerfile
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet restore src/DemoApi/DemoApi.csproj
```

Try it with the Java build: add a new dependency to `pom.xml`, rebuild with and without the cache mount, and compare the times. Note that the cache is **not** part of the image, which is why it is allowed to survive a cache miss.

### 6.5 BuildKit secrets

Secrets must not appear in `ARG`, `ENV` or `COPY`, because they remain readable in the image layers. Prove it to yourself:

```dockerfile
FROM alpine:3.20
ARG TOKEN=my-secret-value
RUN echo "using token" > /dev/null
```

```bash
docker build -t leak-demo --build-arg TOKEN=abc123 .
docker history --no-trunc leak-demo        # where do you find the value?
```

The correct way is a **secret mount**, which exists only during one `RUN` and leaves no trace in the image:

```dockerfile
FROM alpine:3.20
RUN --mount=type=secret,id=demo_token \
    echo "token length: $(wc -c < /run/secrets/demo_token)"
```

```bash
echo -n "abc123" > token.txt
docker build --secret id=demo_token,src=token.txt --progress=plain -t secret-demo .
docker history --no-trunc secret-demo      # the value is not there
```

Delete `token.txt` and never commit it.

### 6.6 Optional: multi-platform builds

An image can contain variants for several CPU architectures (`linux/amd64` for most servers and PCs, `linux/arm64` for Apple Silicon and many cloud machines).

```bash
docker buildx ls
docker buildx build --platform linux/amd64,linux/arm64 -t demo-dotnet:multi .
```

Depending on your Docker setup, you may first have to create a builder (`docker buildx create --use`), and the result of a multi-platform build cannot be loaded into the classic local image store. It stays in the build cache unless you push it. Explain, in your own words, why the .NET and Java images are *portable* across architectures, while a native compiled binary would not be.

### Expected result of Task 6

- Updated Dockerfiles with `ARG`, `LABEL` and a `test` stage, committed.
- The output of `docker image inspect` showing the labels.
- Evidence (command output) that `--target test` fails with a failing test.
- Your observations on the cache mount and on the secret leak versus secret mount.

---

## Task 7: Tagging, publishing and running the image

An image on your laptop is not useful to anyone else. In this task, you will publish it to a **container registry** manually. Automating this step is part of the next exercise.

### 7.1 Tags and digests

A **tag** is a movable pointer. A **digest** is the immutable content hash.

```bash
docker tag demo-dotnet:multistage demo-dotnet:0.1.0
docker tag demo-dotnet:multistage demo-dotnet:0.1
docker tag demo-dotnet:multistage demo-dotnet:latest
docker images demo-dotnet                    # several tags, same IMAGE ID
```

A common scheme, which fits the git tags from Lab 4:

| Tag | Meaning | Changes over time? |
|-----|---------|--------------------|
| `0.1.0` | Exact release (from the git tag `v0.1.0`) | should not |
| `0.1` | Latest patch of the 0.1 line | yes |
| `latest` | The latest release | yes. Avoid in deployments |
| `sha-<commit>` (e.g. `sha-3f2c1ab`) | Exactly the commit it was built from (`git rev-parse --short HEAD`) | no |

### 7.2 Publish to GitHub Container Registry (`ghcr.io`)

1. On GitHub: **Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token**, with the scope **`write:packages`** (and `read:packages`). Copy the token.
2. Log in (the token replaces the password; the image name must be **lowercase**):

```bash
echo "<YOUR_TOKEN>" | docker login ghcr.io -u <github-username> --password-stdin
```

3. Tag with the full registry name and push:

```bash
docker tag demo-dotnet:0.1.0 ghcr.io/<owner>/demo-dotnet:0.1.0
docker push ghcr.io/<owner>/demo-dotnet:0.1.0
docker push ghcr.io/<owner>/demo-dotnet:latest      # after tagging it in the same way
```

4. Find the package under your profile → **Packages**. New packages are **private** by default. Change the visibility to public in the package settings if you want others to pull it, and link it to your repository (this happens automatically if the image has the `org.opencontainers.image.source` label from 6.2).
5. Delete the local image and pull it back from the registry, then run it:

```bash
docker rmi ghcr.io/<owner>/demo-dotnet:0.1.0
docker pull ghcr.io/<owner>/demo-dotnet:0.1.0
docker run --rm -p 8080:8080 ghcr.io/<owner>/demo-dotnet:0.1.0
```

(Docker Hub works the same way: `docker login`, then `docker tag ... <dockerhub-user>/demo-dotnet:0.1.0` and `docker push`.)

### 7.3 Run by digest

```bash
docker image inspect ghcr.io/<owner>/demo-dotnet:0.1.0 --format '{{index .RepoDigests 0}}'
docker run --rm -p 8080:8080 ghcr.io/<owner>/demo-dotnet@sha256:<digest>
```

Push a *different* image under the same tag `0.1.0` (make a change and push again, as an experiment on your demo repository). What happened to the old image, and to the digest? Why is running by digest safer than running by tag?

### 7.4 Running containers (a short reference)

```bash
docker run -d --name demo -p 8080:8080 -e ASPNETCORE_ENVIRONMENT=Development demo-dotnet:0.1.0
docker ps                       # running containers
docker logs -f demo             # follow the logs
docker exec -it demo sh         # a shell inside (not possible in distroless/chiseled images)
docker stop demo && docker rm demo
```

What does `-p 8080:8080` mean (left side, right side)? What happens if the application listens on 8080 in the container, but you publish `-p 9090:8081`?

### Expected result of Task 7

- A public (or accessible) image in `ghcr.io`, with a link to the package page in your report.
- The output of `docker images` with several tags of the same image ID.
- The digest and the answers to the questions in 7.3 and 7.4.

---

## Task 8: Containerize your own project

Apply what you learned to the **project repository** from Lab 1.

1. Write a multi-stage `Dockerfile` for your project's technology stack (from Lab 2). If the stack is neither .NET nor Java (for example Node.js or Python), use the same structure: *dependencies first → source → build → small runtime stage*.
2. Add a `.dockerignore`.
3. The final image must run as a **non-root** user, use the **exec form** for `ENTRYPOINT`, use a **pinned** base image version, and carry the `org.opencontainers.image.*` labels.
4. Build it with `docker build`, run it, and verify that your application responds.
5. Document the build in `docs/docker-build.md`: the exact commands to build, run and tag the image, the image size, the structure of the Dockerfile (a short description of each stage), and the base image you chose and why.
6. In your GitHub Project (Lab 3), create the issues for this work, create a branch from each issue, and merge it with a pull request that uses `Closes #<n>`.

> In the next exercise, the commands from `docs/docker-build.md` become the steps of an automated pipeline. If you can reproduce the build on any machine with the commands you wrote down, automating it is easy.

### Expected result of Task 8

- `Dockerfile` and `.dockerignore` in your project repository, merged through a pull request.
- `docs/docker-build.md`.

---

## Good-practice checklist

| Practice | Why |
|----------|-----|
| Pin the base image version (`:10.0`, `:21-jre`, never `:latest`); even better, pin by digest (`@sha256:...`) | Reproducible builds, no surprise upgrades |
| Multi-stage build; final stage as small as possible | Smaller size, smaller attack surface, faster pull and deploy |
| Dependency files first, source last | Layer cache (Rules 1 and 2) |
| `.dockerignore` | Smaller context, no leaked files, fewer cache invalidations |
| Run as a non-root user | A compromised app does not get root inside the container |
| Exec form for `ENTRYPOINT`/`CMD` | Correct signal handling and shutdown |
| One process per container, configuration through environment variables | The container is replaceable and configurable |
| Combine related shell commands into one `RUN` (`RUN a && b && c`) | Fewer layers; temporary files removed in the same layer are really gone |
| **Never** put secrets in `ARG`, `ENV` or `COPY`; use secret mounts | They remain readable in the image layers and in `docker history` |
| Label the image (`org.opencontainers.image.*`) and tag it with a version | Traceability: which code does this image contain? |

---

## Troubleshooting

| Symptom | Likely cause and fix |
|---------|----------------------|
| `COPY failed: file not found` | The file is outside the build context, or it is excluded in `.dockerignore` |
| The build is always slow | A change high in the Dockerfile invalidates the later layers (Rule 1); `COPY . .` placed before the restore; missing `.dockerignore` |
| `docker: Cannot connect to the Docker daemon` | Docker Desktop is not running (or, on Linux, your user is not in the `docker` group) |
| Container starts and stops immediately | The application crashed: `docker logs <container>`; or the `ENTRYPOINT` points to a wrong file name |
| `curl localhost:8080` refuses the connection | You forgot `-p 8080:8080`, or the app listens on another port inside the container |
| `permission denied` for the app in the container | A file copied into the image is not readable by the non-root user, or the app wants to write to a read-only directory |
| `docker push` says `denied` / `unauthorized` | Not logged in, the token lacks `write:packages`, or the image name is not lowercase |
| `./mvnw: Permission denied` in the build | The executable bit was lost; use `git update-index --chmod=+x mvnw` and commit |
| `no matching manifest for linux/arm64` | The base image has no variant for your CPU; pick another tag or add `--platform` |
| `failed to solve: ... not found` for a base image | A typo in the image or tag; check it on the registry |

---

## Deliverables

- Repository `docker-build-demo` (public) containing both applications, all Dockerfiles from Tasks 3 to 6 and the `.dockerignore` files.
- A report `docs/docker-build-lab.md` in your **project** repository with: the answers to the questions of every task, the filled-in tables (cache behavior and image comparison), the annotated multi-stage Dockerfile, the `docker history` outputs, and the link to the published package.
- `Dockerfile`, `.dockerignore` and `docs/docker-build.md` in your **project** repository (Task 8).

Final checklist:

- [ ] .NET and Java apps built with their native tools; build steps listed
- [ ] Naive Dockerfiles built and analyzed with `docker history`
- [ ] Cache behavior observed and the table filled in
- [ ] `.dockerignore` files created
- [ ] Multi-stage Dockerfiles for .NET and Java, running as non-root
- [ ] Image comparison table filled in with your own measurements
- [ ] `ARG`, `LABEL`, a `test` stage and a cache mount used
- [ ] Secret leak demonstrated, and a secret mount used
- [ ] Image tagged, pushed to `ghcr.io`, pulled and run, also by digest
- [ ] Your project is containerized and documented

---

## Command cheat sheet

| Goal | Command |
|------|---------|
| .NET full build | `dotnet restore && dotnet build -c Release --no-restore && dotnet test -c Release --no-build` |
| .NET publish | `dotnet publish <project> -c Release -o publish` |
| Maven full build | `./mvnw -B verify` |
| Maven without tests | `./mvnw -B package -DskipTests` |
| Build an image | `docker build -t name:tag .` |
| Use another Dockerfile / stage | `docker build -f Dockerfile.naive --target build -t name:tag .` |
| Show every step of a build | `docker build --progress=plain .` |
| Ignore the cache | `docker build --no-cache .` |
| Pass a build argument / a secret | `docker build --build-arg NAME=value .` / `docker build --secret id=x,src=file .` |
| List images and sizes | `docker images` |
| Layers of an image | `docker history name:tag` (`--no-trunc` for full commands) |
| Metadata of an image | `docker image inspect name:tag` |
| Run a container | `docker run --rm -p 8080:8080 name:tag` |
| Logs / shell of a container | `docker logs <container>` / `docker exec -it <container> sh` |
| Tag an image | `docker tag name:tag registry/owner/name:tag` |
| Log in / push / pull | `docker login ghcr.io` / `docker push <image>` / `docker pull <image>` |
| Clean up | `docker system df`, `docker image prune`, `docker builder prune` |

---

## Useful resources

- [Docker Docs – Building best practices](https://docs.docker.com/build/building/best-practices/)
- [Docker Docs – Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Docker Docs – Build cache](https://docs.docker.com/build/cache/)
- [Docker Docs – Build secrets](https://docs.docker.com/build/building/secrets/)
- [Docker Docs – Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [GitHub Docs – Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Microsoft Learn – .NET CLI overview](https://learn.microsoft.com/en-us/dotnet/core/tools/)
- [Maven – Introduction to the build lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)
